require 'fileutils'

MRUBY_VERSION="1.2.0"

file :mruby do
  sh "git clone --depth=1 --branch bignum-1.2 https://github.com/scalone/mruby"
  #sh "curl -L --fail --retry 3 --retry-delay 1 https://github.com/mruby/mruby/archive/1.2.0.tar.gz -s -o - | tar zxf -"
  #FileUtils.mv("mruby-1.2.0", "mruby")
end

APP_NAME=ENV["APP_NAME"] || "cloudwalk"
APP_ROOT=ENV["APP_ROOT"] || Dir.pwd
# avoid redefining constants in mruby Rakefile
mruby_root=File.expand_path(ENV["MRUBY_ROOT"] || "#{APP_ROOT}/mruby")
mruby_config=File.expand_path(ENV["MRUBY_CONFIG"] || "build_config.rb")
ENV['MRUBY_ROOT'] = mruby_root
ENV['MRUBY_CONFIG'] = mruby_config
Rake::Task[:mruby].invoke unless Dir.exist?(mruby_root)
Dir.chdir(mruby_root)
load "#{mruby_root}/Rakefile"

desc "compile binary"
task :compile => [:da_funk, :all] do
  %W(#{mruby_root}/build/x86_64-pc-linux-gnu/bin/#{APP_NAME} #{mruby_root}/build/i686-pc-linux-gnu/#{APP_NAME}").each do |bin|
    sh "strip --strip-unneeded #{bin}" if File.exist?(bin)
  end
end

namespace :test do
  desc "run mruby & unit tests"
  # only build mtest for host
  task :mtest => :compile do
    # in order to get mruby/test/t/synatx.rb __FILE__ to pass,
    # we need to make sure the tests are built relative from mruby_root
    MRuby.each_target do |target|
      # only run unit tests here
      target.enable_bintest = false
      run_test if target.test_enabled?
    end
  end

  def clean_env(envs)
    old_env = {}
    envs.each do |key|
      old_env[key] = ENV[key]
      ENV[key] = nil
    end
    yield
    envs.each do |key|
      ENV[key] = old_env[key]
    end
  end

  desc "run integration tests"
  task :bintest => :compile do
    MRuby.each_target do |target|
      clean_env(%w(MRUBY_ROOT MRUBY_CONFIG)) do
        run_bintest if target.bintest_enabled?
      end
    end
  end
end

desc "Create gem package"
task :gem do
  FileUtils.cd(File.dirname(File.realpath(__FILE__)))
  FileUtils.rm_rf("./exe")
  FileUtils.mkdir_p("./exe")

  files = Dir["mruby/build/**/bin/cloudwalk*"].inject({}) do |hash, f|
    arch = f.match(/mruby\/build\/(.*)\/bin/)[1]
    if arch == "host"
      hash
    else
      if f[-3..-1] ==  "exe"
        hash[f] = "exe/#{arch}.exe"
      else
        hash[f] = "exe/#{arch}"
      end
      hash
    end
  end

  files.each do |from,to|
    FileUtils.cp(from, to)
  end

  FileUtils.chmod 0755, Dir["exe/*"]

  sh "gem build cloudwalk.gemspec"
end

desc "run all tests"
Rake::Task['test'].clear
task :test => ["test:mtest", "test:bintest"]

desc "cleanup"
task :clean do
  sh "rake deep_clean"
end


desc "run all tests"
Rake::Task['test'].clear
task :test => ["test:mtest", "test:bintest"]

desc "cleanup"
task :clean do
  sh "rake deep_clean"
end

desc "Generate da_funk.c from zips"
task :da_funk do
  require "archive/zip"

  FileUtils.cd(File.dirname(File.realpath(__FILE__)))
  FileUtils.cd "./main"
  sh "bundle install"
  sh "bundle exec rake"

  FileUtils.cd "out/main"
  FileUtils.rm_rf "./da_funk"
  Archive::Zip.archive "./da_funk", "./"

  FileUtils.mv "da_funk", "../../../"
  FileUtils.cd "../../../"

  File.open("src/da_funk.c", "w") {|f| f.write("// Automatic generated by rake task\n") }
  sh "xxd -i da_funk >> src/da_funk.c"
end
