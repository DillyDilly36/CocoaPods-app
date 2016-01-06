VERBOSE = !!RakeFileUtils.verbose_flag

RELEASE_PLATFORM = '10.11'

DEPLOYMENT_TARGET = '10.10'

# Ideally this would be deployment target, but
# we use generics which didn't exist in 10.10.
DEPLOYMENT_TARGET_SDK = "MacOSX#{RELEASE_PLATFORM}.sdk"

$build_started_at = Time.now
at_exit do
  min, sec = (Time.now - $build_started_at).divmod(60)
  sec = sec.round
  puts
  puts "Finished in #{min} minutes and #{sec} seconds"
  puts
end

# OpenSSL fails if we set this make configuration through MAKEFLAGS, so we pass
# it to each make invocation seperately.
MAKE_CONCURRENCY = `sysctl hw.physicalcpu`.strip.match(/\d+$/)[0].to_i + 1

ROOT = File.dirname(__FILE__)
PKG_DIR = 'pkg'
DOWNLOAD_DIR = 'downloads'
WORKBENCH_DIR = 'workbench'
DESTROOT = 'destroot'
BUNDLE_DESTROOT = File.join(DESTROOT, 'bundle')
DEPENDENCIES_DESTROOT = File.join(DESTROOT, 'dependencies')

PATCHES_DIR = File.expand_path('patches')
BUNDLE_PREFIX = File.expand_path(BUNDLE_DESTROOT)
DEPENDENCIES_PREFIX = File.expand_path(DEPENDENCIES_DESTROOT)
BUNDLE_ENV = File.join(BUNDLE_PREFIX, 'bin', 'bundle-env')

directory PKG_DIR
directory DOWNLOAD_DIR
directory WORKBENCH_DIR
directory DEPENDENCIES_DESTROOT

# Prefer the SDK of the DEPLOYMENT_TARGET, but otherwise fallback to the current one.
sdk_dir = File.join(`xcrun --show-sdk-platform-path --sdk macosx`.strip, 'Developer/SDKs')
if Dir.entries(sdk_dir).include?(DEPLOYMENT_TARGET_SDK)
  SDKROOT = File.join(sdk_dir, DEPLOYMENT_TARGET_SDK)
else
  SDKROOT = File.expand_path(`xcrun --show-sdk-path --sdk macosx`.strip)
end
unless File.exist?(SDKROOT)
  puts "[!] Unable to find a SDK for the Platform target `macosx`."
  exit 1
end

ORIGINAL_PATH = ENV['PATH']
ENV['PATH'] = "#{File.join(DEPENDENCIES_PREFIX, 'bin')}:/usr/bin:/bin"
ENV['CC'] = '/usr/bin/clang'
ENV['CXX'] = '/usr/bin/clang++'
ENV['CFLAGS'] = "-mmacosx-version-min=#{DEPLOYMENT_TARGET} -isysroot #{SDKROOT}"
ENV['CPPFLAGS'] = "-I#{File.join(DEPENDENCIES_PREFIX, 'include')}"
ENV['LDFLAGS'] = "-L#{File.join(DEPENDENCIES_PREFIX, 'lib')}"

# If we don't create this dir and set the env var, the ncurses configure
# script will simply decide that we don't want any .pc files.
PKG_CONFIG_LIBDIR = File.join(DEPENDENCIES_PREFIX, 'lib/pkgconfig')
ENV['PKG_CONFIG_LIBDIR'] = PKG_CONFIG_LIBDIR

# Defaults to the latest available version or the VERSION env variable.
def install_cocoapods_version
  return @install_cocoapods_version if @install_cocoapods_version
  return @install_cocoapods_version = ENV['VERSION'] if ENV['VERSION']

  Dir.chdir(File.expand_path('~/.cocoapods/repos/master')) do
    execute 'CocoaPods', ['/usr/bin/git', 'pull']
  end

  version_file = File.expand_path('~/.cocoapods/repos/master/CocoaPods-version.yml')
  require 'yaml'
  @install_cocoapods_version = YAML.load(File.read(version_file))['last']
end

# ------------------------------------------------------------------------------
# Package metadata
# ------------------------------------------------------------------------------

PKG_CONFIG_VERSION = '0.28'
PKG_CONFIG_URL = "http://pkg-config.freedesktop.org/releases/pkg-config-#{PKG_CONFIG_VERSION}.tar.gz"

LIBYAML_VERSION = '0.1.6'
LIBYAML_URL = "http://pyyaml.org/download/libyaml/yaml-#{LIBYAML_VERSION}.tar.gz"

ZLIB_VERSION = '1.2.8'
ZLIB_URL = "http://zlib.net/zlib-#{ZLIB_VERSION}.tar.gz"

OPENSSL_VERSION = '1.0.2'
OPENSSL_PATCH = 'd'
OPENSSL_URL = "https://www.openssl.org/source/old/#{OPENSSL_VERSION}/openssl-#{OPENSSL_VERSION}#{OPENSSL_PATCH}.tar.gz"

NCURSES_VERSION = '5.9'
NCURSES_URL = "http://ftpmirror.gnu.org/ncurses/ncurses-#{NCURSES_VERSION}.tar.gz"

READLINE_VERSION = '6.3'
READLINE_URL = "http://ftpmirror.gnu.org/readline/readline-#{READLINE_VERSION}.tar.gz"

RUBY__VERSION = '2.2.3'
RUBY_URL = "http://cache.ruby-lang.org/pub/ruby/2.2/ruby-#{RUBY__VERSION}.tar.gz"

RUBYGEMS_VERSION = '2.5.0'
RUBYGEMS_URL = "https://rubygems.org/downloads/rubygems-update-#{RUBYGEMS_VERSION}.gem"

CURL_VERSION = '7.41.0'
CURL_URL = "http://curl.haxx.se/download/curl-#{CURL_VERSION}.tar.gz"

GIT_VERSION = '2.6.2'
GIT_URL = "https://www.kernel.org/pub/software/scm/git/git-#{GIT_VERSION}.tar.gz"

SCONS_VERSION = '2.3.4'
SCONS_URL = "https://bitbucket.org/scons/scons/get/#{SCONS_VERSION}.tar.gz"

SERF_VERSION = '1.3.8'
SERF_URL = "http://serf.googlecode.com/svn/src_releases/serf-#{SERF_VERSION}.tar.bz2"

SVN_VERSION = '1.8.13'
SVN_URL = "https://archive.apache.org/dist/subversion/subversion-#{SVN_VERSION}.tar.gz"

BZR_VERSION = '2.6.0'
BZR_URL = "https://launchpad.net/bzr/2.6/2.6.0/+download/bzr-#{BZR_VERSION}.tar.gz"

MERCURIAL_VERSION = '3.3.3'
MERCURIAL_URL = "http://mercurial.selenic.com/release/mercurial-#{MERCURIAL_VERSION}.tar.gz"

# ------------------------------------------------------------------------------
# Bundle Build Tools
# ------------------------------------------------------------------------------

def log(group, message)
  $stderr.puts "[#{Time.now.strftime('%T')}] [#{group}] #{message}"
end

def relative_path(path)
  path.start_with?(ROOT) ? path[ROOT.size+1..-1] : path
end

# These changes are so that copy-pasting the logged commands should work.
def log_command(group, command, output_file)
  command_for_presentation = command.map do |component|
    if component.include?('=')
      key, value = component.split('=', 2)
      # Add extra quotes around values of key=value pairs
      %{#{key}="#{value}"}
    else
      component
    end
  end
  wd = Dir.pwd
  if wd == ROOT
    # Make command path relative, if inside `ROOT`
    command_for_presentation[0] = relative_path(command_for_presentation[0])
  else
    # Change working-dir to `wd`
    command_for_presentation.unshift("cd #{relative_path(wd)} &&")
  end
  if output_file
    # Redirect output to `output_file`
    command_for_presentation << '>'
    command_for_presentation << output_file
  end

  log(group, command_for_presentation.join(' '))
end

def execute(group, command, output_file = nil)
  command.map!(&:to_s)
  log_command(group, command, output_file)

  if output_file
    out = File.open(output_file, 'a')
  end
  if VERBOSE
    out ||= $stdout
    err = $stderr
  else
    err = File.open("/tmp/cocoapods-app-bundle-build-#{Process.pid}", 'w+')
    out ||= err
  end
  command << { :out => out, :err => err }

  Process.wait(Process.spawn(*command))
  unless $?.success?
    unless VERBOSE
      out.rewind
      $stderr.puts(out.read)
    end
    exit $?.exitstatus
  end
ensure
  out.close if out && output_file
  err.close if err && !VERBOSE
end

class BundleDependencyTasks
  include Rake::DSL

  def self.define(&block)
    new(&block).tap(&:define_tasks)
  end

  # The URL from where to download the package.
  attr_accessor :url

  # An array of options that should be passed to the `configure` script. The `prefix` is already set.
  attr_accessor :configure

  # A relative path to a file that should exist (in the `WORKBENCH_DIR`) after building the package.
  attr_accessor :artefact_file

  # A relative path to a file that should exist (in the `prefix`) after installing the package.
  attr_accessor :installed_file

  # The installed paths (e.g. `BundleDependencyTasks#installed_path`) that this package depends on.
  attr_accessor :dependencies

  # The `--prefix` value passed to the `configure` script.
  attr_accessor :prefix

  def initialize
    @dependencies = []
    @configure = []
    yield self
  end

  def define_tasks
    define_download_task
    define_unpack_task
    define_build_task
    define_install_task
  end

  def package_name
    File.basename(@url).split('.tar.').first
  end

  def execute(*command)
    super(package_name, command)
  end

  def downloaded_file
    File.join(DOWNLOAD_DIR, File.basename(@url))
  end

  def download_task
    execute '/usr/bin/curl', '-sSL', @url, '-o', downloaded_file
  end

  def define_download_task
    file(downloaded_file => DOWNLOAD_DIR) { download_task }
  end

  def build_dir
    File.join(WORKBENCH_DIR, package_name)
  end

  def unpack_command
    ['/usr/bin/tar', '-zxvf', downloaded_file, '-C', WORKBENCH_DIR]
  end

  def unpack_task
    execute *unpack_command
  end

  def define_unpack_task
    directory(build_dir => [downloaded_file, WORKBENCH_DIR]) { unpack_task }
  end

  def artefact_path
    File.join(build_dir, @artefact_file)
  end

  def build_command
    ['/usr/bin/make', '-j', MAKE_CONCURRENCY]
  end

  def build_task
    Dir.chdir(build_dir) do
      execute '/bin/sh', 'configure', '--prefix', @prefix, *@configure
      execute *build_command
    end
  end

  def define_build_task
    dependencies = @dependencies + [build_dir]
    file(artefact_path => dependencies) { build_task }
  end

  def installed_path
    File.join(relative_path(@prefix), @installed_file)
  end

  def install_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/make', 'install'
    end
  end

  def define_install_task
    file(installed_path => artefact_path) { install_task }
  end
end

class PythonSetupTasks < BundleDependencyTasks
  def self.python_version
    @python_version ||= `/usr/bin/python --version 2>&1`.match(/\d\.\d/)[0]
  end

  def artefact_script=(script_name)
    self.artefact_file = File.join('build', "scripts-#{self.class.python_version}", script_name)
  end

  def build_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/python', 'setup.py', 'build'
    end
  end

  def install_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/python', 'setup.py', 'install', '--prefix', BUNDLE_PREFIX
    end
  end
end

GEM_HOME = File.join(BUNDLE_DESTROOT, 'lib/ruby/gems', RUBY__VERSION.sub(/\d+$/, '0'))

def install_gem(name, version = nil, group = 'Gems')
  execute group, [BUNDLE_ENV, 'gem', 'install', name, ("--version=#{version}" if version), '--no-document', '--env-shebang'].compact
end

# ------------------------------------------------------------------------------
# pkg-config
# ------------------------------------------------------------------------------

class PkgConfigTasks < BundleDependencyTasks
  def install_task
    super
    mkdir_p PKG_CONFIG_LIBDIR
  end
end

pkg_config_tasks = PkgConfigTasks.define do |t|
  t.url            = PKG_CONFIG_URL
  t.artefact_file  = 'pkg-config'
  t.installed_file = 'bin/pkg-config'
  t.prefix         = DEPENDENCIES_PREFIX
  t.configure      = %w{ --enable-static --with-internal-glib }
end

installed_pkg_config = pkg_config_tasks.installed_path

# ------------------------------------------------------------------------------
# YAML
# ------------------------------------------------------------------------------

yaml_tasks = BundleDependencyTasks.define do |t|
  t.url            = LIBYAML_URL
  t.artefact_file  = 'src/.libs/libyaml.a'
  t.installed_file = 'lib/libyaml.a'
  t.configure      = %w{ --disable-shared }
  t.prefix         = DEPENDENCIES_PREFIX
  t.dependencies   = [installed_pkg_config]
end

installed_yaml = yaml_tasks.installed_path

# ------------------------------------------------------------------------------
# ZLIB
# ------------------------------------------------------------------------------

zlib_tasks = BundleDependencyTasks.define do |t|
  t.url            = ZLIB_URL
  t.artefact_file  = 'libz.a'
  t.installed_file = 'lib/libz.a'
  t.configure      = %w{ --static }
  t.prefix         = DEPENDENCIES_PREFIX
  t.dependencies   = [installed_pkg_config]
end

installed_zlib = zlib_tasks.installed_path

# ------------------------------------------------------------------------------
# OpenSSL
# ------------------------------------------------------------------------------

class OpenSSLTasks < BundleDependencyTasks
  def build_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/perl', 'Configure', "--prefix=#{DEPENDENCIES_PREFIX}", 'no-shared', 'zlib', 'darwin64-x86_64-cc'
      # OpenSSL needs to be build with at max 1 process
      execute '/usr/bin/make', '-j', '1'
    end
    # Seems to be a OpenSSL bug in the pkg-config, as libz is required when
    # linking libssl, otherwise Ruby's openssl ext will fail to configure.
    # So add it ourselves.
    %w( libcrypto.pc libssl.pc ).each do |pc_filename|
      pc_file = File.join(build_dir, pc_filename)
      log(package_name, "Patching: #{pc_file}")
      original_content = File.read(pc_file)
      content = original_content.sub(/Libs:/, 'Libs: -lz')
      if original_content == content
        raise "[!] Did not patch anything in: #{pc_file}"
      end
      File.open(pc_file, 'w') { |f| f.write(content) }
    end
  end
end

openssl_tasks = OpenSSLTasks.define do |t|
  t.url            = OPENSSL_URL
  t.artefact_file  = 'libssl.a'
  t.installed_file = 'lib/libssl.a'
  t.prefix         = DEPENDENCIES_PREFIX
  t.dependencies   = [installed_pkg_config, installed_zlib]
end

installed_openssl = openssl_tasks.installed_path

# ------------------------------------------------------------------------------
# ncurses
# ------------------------------------------------------------------------------

class NCursesTasks < BundleDependencyTasks
  def unpack_task
    super
    Dir.chdir(build_dir) do
      execute '/usr/bin/patch', '-p', '1', '-i', File.join(PATCHES_DIR, 'ncurses.diff')
    end
  end
end

ncurses_tasks = NCursesTasks.define do |t|
  t.url            = NCURSES_URL
  t.artefact_file  = 'lib/libncurses.a'
  t.installed_file = 'lib/libncurses.a'
  t.prefix         = DEPENDENCIES_PREFIX
  t.configure      = %w{ --without-shared --enable-getcap  --with-ticlib --with-termlib --disable-leaks --without-debug --enable-pc-files --with-pkg-config }
  t.dependencies   = [installed_pkg_config]
end

installed_ncurses = ncurses_tasks.installed_path

# ------------------------------------------------------------------------------
# Readline
# ------------------------------------------------------------------------------

readline_tasks = BundleDependencyTasks.define do |t|
  t.url            = READLINE_URL
  t.artefact_file  = 'libreadline.a'
  t.installed_file = 'lib/libreadline.a'
  t.prefix         = DEPENDENCIES_PREFIX
  t.configure      = %w{ --disable-shared --with-curses }
  t.dependencies   = [installed_pkg_config, installed_ncurses]
end

installed_readline = readline_tasks.installed_path

# ------------------------------------------------------------------------------
# Ruby
# ------------------------------------------------------------------------------

class RubyTasks < BundleDependencyTasks
  attr_accessor :installed_libruby_path

  def define_install_libruby_task
    file installed_libruby_path => artefact_path do
      cp artefact_path, installed_libruby_path
      %w{ bigdecimal date/date_core.a pathname stringio }.each do |ext|
        ext = "#{ext}/#{ext}.a" unless File.extname(ext) == '.a'
        execute '/usr/bin/libtool', '-static', '-o', installed_libruby_path, installed_libruby_path, File.join(build_dir, 'ext', ext)
      end
    end
  end

  def define_tasks
    super
    define_install_libruby_task
  end
end

ruby_tasks = RubyTasks.define do |t|
  t.url            = RUBY_URL
  t.artefact_file  = 'libruby-static.a'
  t.installed_file = 'bin/ruby'
  t.prefix         = BUNDLE_PREFIX
  t.configure      = %w{ --enable-load-relative --disable-shared --with-static-linked-ext --disable-install-doc --with-out-ext=,dbm,gdbm,sdbm,dl/win32,fiddle/win32,tk/tkutil,tk,win32ole,-test-/win32/dln,-test-/win32/fd_setsize,-test-/win32/dln/empty }
  t.dependencies   = [installed_pkg_config, installed_yaml, installed_openssl]

  t.installed_libruby_path = File.join('app', 'CPReflectionService', 'libruby+exts.a')
end

installed_ruby = ruby_tasks.installed_path
installed_ruby_static_lib = ruby_tasks.installed_libruby_path

# ------------------------------------------------------------------------------
# bundle-env
# ------------------------------------------------------------------------------

installed_env_script = File.join(BUNDLE_DESTROOT, 'bin/bundle-env')
file installed_env_script do
  log 'bundle-env', 'Installing'
  cp 'bundle-env', installed_env_script
  chmod '+x', installed_env_script
end

# ------------------------------------------------------------------------------
# Gems
# ------------------------------------------------------------------------------

class RubyGemsTasks < BundleDependencyTasks
  def package_name
    File.basename(@url, '.gem')
  end
end

rubygems_tasks = RubyGemsTasks.new { |t| t.url = RUBYGEMS_URL }.tap(&:define_download_task)
rubygems_gem = rubygems_tasks.downloaded_file

rubygems_update_dir = File.join(GEM_HOME, 'gems', rubygems_tasks.package_name)
directory rubygems_update_dir => [installed_ruby, installed_env_script, rubygems_gem] do
  install_gem(rubygems_gem, nil, rubygems_tasks.package_name)
  execute(rubygems_tasks.package_name, [BUNDLE_ENV, 'update_rubygems'])
  # Fix shebang of `gem` bin to use bundled Ruby.
  bin = File.join(BUNDLE_DESTROOT, 'bin/gem')
  log(rubygems_tasks.package_name, "Patching: #{bin}")
  lines = File.read(bin).split("\n")
  lines[0] = '#!/usr/bin/env ruby'
  File.open(bin, 'w') { |f| f.write(lines.join("\n")) }
  chmod '+x', bin
end

# ------------------------------------------------------------------------------
# CocoaPods Gems
# ------------------------------------------------------------------------------

installed_pod_bin = File.join(BUNDLE_DESTROOT, 'bin/pod')
file installed_pod_bin => rubygems_update_dir do
  install_gem 'cocoapods', install_cocoapods_version
end

plugin = 'cocoapods-plugins-install'
$:.unshift "#{plugin}/lib"
require "#{plugin}/gem_version"
plugin_with_version = "#{plugin}-#{CocoapodsPluginsInstall::VERSION}"

installed_cocoapods_plugins_install = File.join(GEM_HOME, 'gems', plugin_with_version)
directory installed_cocoapods_plugins_install => installed_pod_bin do
  Dir.chdir(plugin) do
    execute 'Gems', [BUNDLE_ENV, 'gem', 'build', "#{plugin}.gemspec"]
  end
  install_gem "#{plugin}/#{plugin_with_version}.gem"
end

# ------------------------------------------------------------------------------
# Third-party gems
# ------------------------------------------------------------------------------

# Note, this assumes its being build on the latest OS X version.
installed_osx_gems = []
Dir.glob('/System/Library/Frameworks/Ruby.framework/Versions/[0-9]*/usr/lib/ruby/gems/*/specifications/*.gemspec').each do |gemspec|
  # We have to make some file that does not contain any version information, otherwise we'd first have to query rubygems
  # for the available versions, which is going to take a long time.
  installed_gem = File.join(GEM_HOME, 'specifications', "#{File.basename(gemspec, '.gemspec').split('-')[0..-2].join('-')}.CocoaPods-app.installed")
  installed_osx_gems << installed_gem
  file installed_gem => rubygems_update_dir do
    suppress_upstream = false

    require 'rubygems/specification'
    gem = Gem::Specification.load(gemspec)
    # First install the exact same version that Apple included in OS X.
    case gem.name
    when 'libxml-ruby'
      # libxml-ruby-2.6.0 has an extconf.rb that depends on old behavior where `RbConfig` was available as `Config`.
      install_gem(File.join(PATCHES_DIR, "#{File.basename(gemspec, '.gemspec')}.gem"))
    when 'sqlite3'
      # sqlite3-1.3.7 depends on BigDecimal header from before BigDecimal was made into a gem. I doubt anybody really
      # uses sqlite for CocoaPods dependencies anyways, so just skip this old version.
    when 'nokogiri'
      # nokogiri currently has a design flaw that results in its build
      # failing every time unless I manually patch extconf.rb. I have
      # included a patched copy of nokogiri in the patches/ directory.
      # Until this is remedied, I cannot install the upstream version
      # of nokogiri.
      install_gem(File.join(PATCHES_DIR, "#{File.basename(gemspec, '.gemspec')}.gem"))
      suppress_upstream = true
    else
      install_gem(gem.name, gem.version)
    end
    # Now install the latest version of the gem.
    install_gem(gem.name) unless suppress_upstream
    # Create our nonsense file that's only used to track whether or not the gems were installed.
    touch installed_gem
  end
end

# ------------------------------------------------------------------------------
# cURL
# ------------------------------------------------------------------------------

curl_tasks = BundleDependencyTasks.define do |t|
  t.url            = CURL_URL
  t.artefact_file  = 'lib/.libs/libcurl.a'
  t.installed_file = 'lib/libcurl.a'
  t.prefix         = DEPENDENCIES_PREFIX
  t.configure      = %w{ --disable-shared --enable-static }
  t.dependencies   = [installed_pkg_config, installed_openssl, installed_zlib]
end

installed_libcurl = curl_tasks.installed_path

# ------------------------------------------------------------------------------
# Git
# ------------------------------------------------------------------------------

class GitTasks < BundleDependencyTasks
  def build_command
    super + ['V=1']
  end

  def install_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/env', 'NO_INSTALL_HARDLINKS=1', '/usr/bin/make', 'install'
    end
    # Even after using the NO_INSTALL_HARDLINKS env var, `bin/git*` is still hardlinked to
    # `libexec/git-core/git*`.
    bin = File.join(BUNDLE_DESTROOT, 'bin')
    files = Dir.glob(File.join(bin, 'git*')).reject { |f| File.symlink?(f) }
    Dir.chdir(bin) do
      files.each do |file|
        filename = File.basename(file)
        rm filename
        ln_s "../libexec/git-core/#{filename}", filename
      end
    end
  end
end

git_tasks = GitTasks.define do |t|
  t.url            = GIT_URL
  t.artefact_file  = 'git'
  t.installed_file = 'bin/git'
  t.prefix         = BUNDLE_PREFIX
  t.configure      = ['--without-tcltk', %{LDFLAGS=-L '#{DEPENDENCIES_PREFIX}/lib' -lssl -lcrypto -lz -lcurl -lldap}, %{CPPFLAGS=-I '#{DEPENDENCIES_PREFIX}/include'}]
  t.dependencies   = [installed_pkg_config, installed_openssl, installed_zlib]
end

installed_git = git_tasks.installed_path

# ------------------------------------------------------------------------------
# Scons
# ------------------------------------------------------------------------------

class SconsTasks < BundleDependencyTasks
  def define_tasks
    define_download_task
    define_unpack_task
  end

  def package_name
    "scons-#{SCONS_VERSION}"
  end

  def downloaded_file
    # Don’t download archive as just VERSION.tar.gz
    File.join(DOWNLOAD_DIR, "#{package_name}.tar.gz")
  end

  def unpack_command
    command = super
    command[-1] = build_dir
    # Ignore the root dir which is scons-scons-SHA
    command + %w{ --strip 1 }
  end

  def unpack_task
    mkdir_p build_dir
    super
  end
end

scons_tasks = SconsTasks.define do |t|
  t.url           = SCONS_URL
  t.artefact_file = 'src/script/scons.py'
end

installed_scons = scons_tasks.build_dir
scons_bin = scons_tasks.artefact_path

# ------------------------------------------------------------------------------
# SERF
# ------------------------------------------------------------------------------

class SerfTasks < BundleDependencyTasks
  attr_accessor :scons_bin

  def unpack_command
    command = super
    # bzip instead of gzip
    command[1].tr!('z', 'j')
    command
  end

  def build_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/python', scons_bin, "PREFIX=#{DEPENDENCIES_PREFIX}", "OPENSSL=#{DEPENDENCIES_PREFIX}", "ZLIB=#{DEPENDENCIES_PREFIX}", "CPPFLAGS=-I #{SDKROOT}/usr/include/apr-1"
    end
    # Seems to be a SERF bug in the pkg-config, as libssl, libcrypto, and libz is
    # required when linking libssl, otherwise svn will fail to build with our
    # OpenSSl. So add it ourselves.
    pc_file = File.join(build_dir, 'serf-1.pc')
    log(package_name, "Patching: #{pc_file}")
    original_content = File.read(pc_file)
    content = original_content.sub('Libs: -L${libdir}', 'Libs: -L${libdir} -lssl -lcrypto -lz')
    if original_content == content
      raise "[!] Did not patch anything in: #{pc_file}"
    end
    File.open(pc_file, 'w') { |f| f.write(content) }
  end

  def install_task
    Dir.chdir(build_dir) do
      execute '/usr/bin/python', scons_bin, 'install'
    end
    rm Dir.glob(File.join(DEPENDENCIES_DESTROOT, 'lib', '*.dylib'))
  end
end

serf_task = SerfTasks.define do |t|
  t.scons_bin      = File.expand_path(scons_bin)
  t.url            = SERF_URL
  t.artefact_file  = 'libserf-1.a'
  t.installed_file = 'lib/libserf-1.a'
  t.prefix         = DEPENDENCIES_PREFIX
  t.configure      = %w{ --disable-shared --with-curses }
  t.dependencies   = [installed_pkg_config, installed_ncurses, installed_scons]
end

installed_serf = serf_task.installed_path

# ------------------------------------------------------------------------------
# Subversion
# ------------------------------------------------------------------------------

class SVNTasks < BundleDependencyTasks
  def unpack_task
    super
    Dir.chdir(build_dir) do
      execute '/usr/bin/patch', '-p', '0', '-i', File.join(PATCHES_DIR, 'svn-configure.diff')
    end
  end
end

svn_tasks = SVNTasks.define do |t|
  t.url            = SVN_URL
  t.artefact_file  = 'subversion/svn/svn'
  t.installed_file = 'bin/svn'
  t.prefix         = BUNDLE_PREFIX
  t.configure      = %w{ --disable-shared --enable-all-static --with-serf --without-apxs --without-jikes --without-swig } + ["CPPFLAGS=-I '#{SDKROOT}/usr/include/apr-1'"]
  t.dependencies   = [installed_pkg_config, installed_serf, installed_libcurl]
end

installed_svn = svn_tasks.installed_path

# ------------------------------------------------------------------------------
# Mercurial
# ------------------------------------------------------------------------------

mercurial_tasks = PythonSetupTasks.define do |t|
  t.url             = MERCURIAL_URL
  t.artefact_script = 'hg'
  t.installed_file  = 'bin/hg'
  t.prefix          = BUNDLE_PREFIX
  t.dependencies    = [installed_libcurl]
end

installed_mercurial = mercurial_tasks.installed_path

# ------------------------------------------------------------------------------
# Bazaar
# ------------------------------------------------------------------------------

bzr_tasks = PythonSetupTasks.define do |t|
  t.url             = BZR_URL
  t.artefact_script = 'bzr'
  t.installed_file  = 'bin/bzr'
  t.prefix          = BUNDLE_PREFIX
  t.dependencies    = [installed_pkg_config, installed_libcurl]
end

installed_bzr = bzr_tasks.installed_path

# ------------------------------------------------------------------------------
# Root Certificates
# ------------------------------------------------------------------------------

installed_cacert = File.join(BUNDLE_DESTROOT, 'share/cacert.pem')
file installed_cacert do
  %w{ /Library/Keychains/System.keychain /System/Library/Keychains/SystemRootCertificates.keychain }.each do |keychain|
    execute 'Certificates', ['/usr/bin/security', 'find-certificate', '-a', '-p', keychain], installed_cacert
  end
end

# ------------------------------------------------------------------------------
# Bundle tasks
# ------------------------------------------------------------------------------

namespace :bundle do
  task :build_tools => [
    installed_ruby,
    installed_pod_bin,
    installed_cocoapods_plugins_install,
    installed_git,
    installed_svn,
    installed_bzr,
    installed_mercurial,
    installed_env_script,
    installed_cacert,
  ].concat(installed_osx_gems)

  task :remove_unneeded_files => :build_tools do
    remove_if_existant = lambda do |*paths|
      paths.each do |path|
        rm_rf(path) if File.exist?(path)
      end
    end
    if VERBOSE
      puts
      puts "Before clean:"
      sh "du -hs #{BUNDLE_DESTROOT}"
    end
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, 'bin/svn[a-z]*'))
    remove_if_existant.call *FileList[File.join(BUNDLE_DESTROOT, 'lib/**/*.{,l}a')]
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, 'lib/ruby/gems/**/*.o'))
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, 'lib/ruby/gems/*/cache'))
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, '**/man[0-9]'))
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, '**/.DS_Store'))
    remove_if_existant.call File.join(BUNDLE_DESTROOT, 'include/subversion-1')
    remove_if_existant.call File.join(BUNDLE_DESTROOT, 'man')
    remove_if_existant.call File.join(BUNDLE_DESTROOT, 'share/gitweb')
    remove_if_existant.call File.join(BUNDLE_DESTROOT, 'share/man')
    remove_if_existant.call *Dir.glob(File.join(BUNDLE_DESTROOT, 'lib/python*/site-packages/bzrlib/tests'))
    # Remove all uncompiled `py` files.
    Dir.glob(File.join(BUNDLE_DESTROOT, 'lib/python*/**/*.pyc')).each do |pyc|
      remove_if_existant.call pyc[0..-2]
    end
    # TODO clean Ruby stdlib
    if VERBOSE
      puts "After clean:"
      sh "du -hs #{BUNDLE_DESTROOT}"
    end
  end

  desc "Verifies that no binaries in the bundle link to incorrect dylibs"
  task :verify_linkage => :remove_unneeded_files do
    skip = %w( .h .rb .py .pyc .tmpl .pem .png .ttf .css .rhtml .js .sample )
    Dir.glob(File.join(BUNDLE_DESTROOT, '**/*')).each do |path|
      next if File.directory?(path)
      next if skip.include?(File.extname(path))
      linkage = `otool -arch x86_64 -L '#{path}'`.strip
      unless linkage.include?('is not an object file')
        linkage = linkage.split("\n")[1..-1]

        puts
        puts "Linkage of `#{path}`:"
        puts linkage

        good = linkage.grep(%r{^\s+(/System/Library/Frameworks/|/usr/lib/)})
        bad = linkage - good
        unless bad.empty?
          puts
          puts "[!] Bad linkage found in `#{path}`:"
          puts bad
          exit 1
        end
      end
    end
  end

  desc "Test bundle"
  task :test => :build_tools do
    test_dir = 'tmp'
    rm_rf test_dir
    mkdir_p test_dir
    cp 'test/Podfile', test_dir
    Dir.chdir(test_dir) do
      execute 'Test', [BUNDLE_ENV, 'pod', 'install', '--no-integrate', '--verbose']
    end
  end

  desc "Ensure Submodules are downloaded"
  task :submodules do
    execute 'Submodules', ['/usr/bin/git', 'submodule', 'update', '--init', '--recursive']
  end

  desc "Build complete dist bundle"
  task :build => [:build_tools, :remove_unneeded_files]

  namespace :clean do
    task :build do
      rm_rf WORKBENCH_DIR
    end

    task :downloads do
      rm_rf DOWNLOAD_DIR
    end

    task :destroot do
      rm_rf DESTROOT
    end

    desc "Clean build and destroot artefacts"
    task :artefacts => [:build, :destroot]

    desc "Clean all artefacts, including downloads"
    task :all => [:artefacts, :downloads]
  end
end

# ------------------------------------------------------------------------------
# RubyCocoa
# ------------------------------------------------------------------------------

built_rubycocoa = 'app/RubyCocoa/framework/build/Default/RubyCocoa.framework/Versions/A/RubyCocoa'
file built_rubycocoa => [installed_ruby, installed_env_script] do
  Dir.chdir('app/RubyCocoa') do
    execute 'RubyCocoa', [BUNDLE_ENV, 'ruby', 'install.rb', 'config', '--target-archs=x86_64', '--build-as-embeddable=yes']
    execute 'RubyCocoa', [BUNDLE_ENV, 'ruby', 'install.rb', 'setup']
  end
end

# ------------------------------------------------------------------------------
# CocoaPods.app
# ------------------------------------------------------------------------------

XCODEBUILD_COMMAND = %w{ /usr/bin/xcodebuild -workspace CocoaPods.xcworkspace -scheme CocoaPods -configuration Release }

namespace :app do
  desc 'Updates the Info.plist of the application to reflect the CocoaPods version'
  task :update_version do
    info_plist = File.expand_path('app/CocoaPods/Supporting Files/Info.plist')
    execute 'App', ['/usr/libexec/PlistBuddy', '-c', "Set :CFBundleShortVersionString #{install_cocoapods_version}", info_plist]
    execute 'App', ['/usr/libexec/PlistBuddy', '-c', "Set :CFBundleVersion #{install_cocoapods_version}", info_plist]
  end

  desc 'Prepare all prerequisites for building the app'
  task :prerequisites => ['bundle:submodules', 'bundle:build', installed_ruby_static_lib, built_rubycocoa, :update_version]

  desc 'Build release version of application'
  task :build => :prerequisites do
    Dir.chdir('app') do
      execute 'App', XCODEBUILD_COMMAND + ["MACOSX_DEPLOYMENT_TARGET=#{DEPLOYMENT_TARGET}", "SDKROOT=#{SDKROOT}", "CODE_SIGN_IDENTITY=Developer ID Application", 'build']
    end
  end

  desc 'Clean'
  task :clean do
    Dir.chdir('app') do
      execute 'App', XCODEBUILD_COMMAND + ['clean']
    end
  end
end

# ------------------------------------------------------------------------------
# Release tasks
# ------------------------------------------------------------------------------

def github_access_token
  File.read('.github_access_token').strip rescue nil
end

namespace :release do
  task :clean => ['bundle:clean:all', 'app:clean']

  desc "Perform a full build of the bundle and app"
  task :build => ['bundle:build', 'bundle:verify_linkage', 'bundle:test', 'app:build', PKG_DIR] do
    output = `#{XCODEBUILD_COMMAND} -showBuildSettings | grep -w BUILT_PRODUCTS_DIR`.strip
    build_dir = output.split('= ').last
    # TODO use this once OS X supports xz out of the box.
    #tarball = File.expand_path(File.join(PKG_DIR, "CocoaPods.app-#{install_cocoapods_version}.tar.xz"))
    #sh "cd '#{build_dir}' && tar cfJ '#{tarball}' CocoaPods.app"
    tarball = File.expand_path(File.join(PKG_DIR, "CocoaPods.app-#{install_cocoapods_version}.tar.bz2"))
    Dir.chdir(build_dir) do
      execute 'App', ['/usr/bin/tar', 'cfj', tarball, 'CocoaPods.app']
    end
  end

  desc "Create a clean build"
  task :cleanbuild => [:clean, :build]

  desc "Upload release"
  task :upload => [] do
    tarball = File.expand_path(File.join(PKG_DIR, "CocoaPods.app-#{install_cocoapods_version}.tar.bz2"))
    sha = `shasum -a 256 -b '#{tarball}'`.split(' ').first

    require 'net/http'
    require 'json'
    require 'rest'

    github_headers = {
      'Content-Type' => 'application/json',
      'User-Agent' => 'runscope/0.1,segiddins',
      'Accept' => 'application/json',
    }

    response = REST.post("https://api.github.com/repos/CocoaPods/CocoaPods-app/releases?access_token=#{github_access_token}",
                         {tag_name: install_cocoapods_version, name: install_cocoapods_version}.to_json,
                         github_headers)

    tarball_name = File.basename(tarball)

    upload_url = JSON.load(response.body)['upload_url'].gsub('{?name,label}', "?name=#{tarball_name}&Content-Type=application/x-tar&access_token=#{github_access_token}")
    response = REST.post(upload_url, File.read(tarball, :mode => 'rb'), github_headers)
    tarball_download_url = JSON.load(response.body)['browser_download_url']

    puts
    puts "Make a PR to https://github.com/CocoaPods/CocoaPods-app/blob/master/homebrew-cask " \
         "updating the version to #{install_cocoapods_version} and the sha to #{sha}"
    puts
  end
end

desc "Create a clean release build for distribution"
task :release do
  unless `sw_vers -productVersion`.strip.split('.').first(2).join('.') == RELEASE_PLATFORM
    puts "[!] A release build must be performed on the latest OS X version to ensure all the gems that Apple includes " \
         "in its OS will be bundled."
    exit 1
  end
  unless File.basename(SDKROOT) == DEPLOYMENT_TARGET_SDK
    puts "[!] Unable to find the SDK for the deployment target `#{DEPLOYMENT_TARGET}`, which is required to create a " \
         "distribution release."
    exit 1
  end
  unless github_access_token
    puts "[!] You have not provided a github access token via `.github_access_token`, " \
         'so a GitHub release cannot be made.'
    exit 1
  end
  Rake::Task['release:cleanbuild'].invoke
  Rake::Task['release:upload'].invoke
end
