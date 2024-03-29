$: << File.expand_path( '../lib/', __FILE__ )
require 'rake/clean'
require 'fileutils'
require 'find'
require 'rspec/core/rake_task'
require 'rubygems'
require 'simp/rake'
require 'simp/rake/ci'
require 'simp/test/version'

# Create pkg:* and simp:ci_* Rake tasks
Simp::Rake::Pkg.new(File.dirname(__FILE__))
Simp::Rake::Ci.new(File.dirname(__FILE__))

@package='simp-test'
@rakefile_dir=File.dirname(__FILE__)

CLEAN.include "#{@package}-*.gem"
CLEAN.include 'coverage'
CLEAN.include 'dist'
CLEAN.include 'pkg'
Find.find( @rakefile_dir ) do |path|
  if File.directory? path
    CLEAN.include path if File.basename(path) == 'tmp'
  else
    Find.prune
  end
end


desc 'Ensure gemspec-safe permissions on all files'
task :chmod do
  gemspec = File.expand_path( "#{@package}.gemspec", @rakefile_dir ).strip
  spec = Gem::Specification::load( gemspec )
  spec.files.each do |file|
    FileUtils.chmod 'go=r', file unless File.symlink?(file)
  end
end

desc 'special notes about these rake commands'
task :help do
  puts %Q{
== environment variables ==
SIMP_RPM_BUILD     when set, alters the gem produced by pkg:gem to be RPM-safe.
                   'pkg:gem' sets this automatically.
  }
end

desc "Run spec tests"
RSpec::Core::RakeTask.new(:spec) do |t|
  t.rspec_opts = ['--color']
  t.exclude_pattern = '**/{acceptance,fixtures,files}/**/*_spec.rb'
  t.pattern = 'spec/{lib,bin}/**/*_spec.rb'
end

namespace :pkg do
  @specfile_template = "rubygem-#{@package}.spec.template"
  @specfile          = "build/rubygem-#{@package}.spec"

  # ----------------------------------------
  # DO NOT UNCOMMENT THIS: the spec file requires a lot of tweaking
  # ----------------------------------------
  #  desc "generate RPM spec file for #{@package}"
  #  task :spec => [:clean, :gem] do
  #    Dir.glob("pkg/#{@package}*.gem") do |pkg|
  #      sh %Q{gem2rpm -t "#{@specfile_template}" "#{pkg}" > "#{@specfile}"}
  #    end
  #  end

  desc "build rubygem package for #{@package}"
  task :gem => :chmod do
    gem_dirs = [@rakefile_dir]
    gem_dirs += Dir.glob('ext/gems/*')

    gem_dirs.each do |gem_dir|
      Dir.chdir gem_dir do
        Dir['*.gemspec'].each do |spec_file|
          # highline optional (development-related) gems require cmake, so
          # exclude them in the bundle
          opts = (File.basename(gem_dir) == 'highline') ? '--without=code_quality' : ''
          cmd = %Q{SIMP_RPM_BUILD=1 bundle exec gem build "#{spec_file}" &> /dev/null}
          require 'bundler'
          if ::Bundler.respond_to?(:with_unbundled_env)
            # Use Bundler 2.x API
            ::Bundler.with_unbundled_env do
              %x{bundle install #{opts}}
              sh cmd
            end
          else
            # Use deprecated Bundler 1.x API
            ::Bundler.with_clean_env do
              %x{bundle install #{opts}}
              sh cmd
            end
          end

          FileUtils.mkdir_p 'dist'
          FileUtils.mv Dir.glob('*.gem'), File.join(@rakefile_dir, 'dist')
        end
      end
    end
  end


  desc "build and install rubygem package for #{@package}"
  task :install_gem => [:clean, :gem] do
    Dir.chdir @rakefile_dir
    Dir.glob("dist/#{@package}*.gem") do |pkg|
      sh %Q{bundle exec gem install #{pkg}}
    end
  end

  desc 'ensure simp cli Ruby version matches its RPM spec file version'
  task :validate_ruby_version do
    basedir = File.dirname(__FILE__)
    info, changelogs = Simp::RelChecks::load_and_validate_changelog(basedir, false)
    spec_file_version = Gem::Version.new(info.version)
    ruby_version = Gem::Version.new(Simp::Cli::VERSION)
    if spec_file_version != ruby_version
      fail("ERROR: Version mismatch: " +
        " spec file version = #{spec_file_version}," +
        " version.rb version = #{ruby_version}")
    end
  end

  Rake::Task[:rpm].prerequisites.unshift(:gem)

  Rake::Task[:compare_latest_tag].enhance [:validate_ruby_version]
end

# Make sure gem packages are built because they will be installed in
# the acceptance test
# vim: syntax=ruby
