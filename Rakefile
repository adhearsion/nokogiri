# -*- ruby -*-

require 'rubygems'
require 'hoe'

kind = Config::CONFIG['DLEXT']

LIB_DIR = File.expand_path(File.join(File.dirname(__FILE__), 'lib'))
$LOAD_PATH << LIB_DIR

GENERATED_PARSER = "lib/nokogiri/css/generated_parser.rb"
GENERATED_TOKENIZER = "lib/nokogiri/css/generated_tokenizer.rb"

EXT = "ext/nokogiri/native.#{kind}"

require 'nokogiri/version'

HOE = Hoe.new('nokogiri', Nokogiri::VERSION) do |p|
  p.developer('Aaron Patterson', 'aaronp@rubyforge.org')
  p.clean_globs = [
    'ext/nokogiri/Makefile',
    'ext/nokogiri/*.{o,so,bundle,a,log,dll}',
    'ext/nokogiri/conftest.dSYM',
    GENERATED_PARSER,
    GENERATED_TOKENIZER,
    'cross',
  ]
  p.spec_extras = { :extensions => ["Rakefile"] }
  p.extra_deps = ["rake"]
end

namespace :gem do
  task :spec do
    File.open("#{HOE.name}.gemspec", 'w') do |f|
      HOE.spec.version = "#{HOE.version}.#{Time.now.strftime("%Y%m%d%H%M%S")}"
      f.write(HOE.spec.to_ruby)
    end
  end
end

desc "Run code-coverage analysis"
task :coverage do
  rm_rf "coverage"
  sh "rcov -x Library -I lib:test #{Dir[*HOE.test_globs].join(' ')}"
end

file GENERATED_PARSER => "lib/nokogiri/css/parser.y" do |t|
  sh "racc -o #{t.name} #{t.prerequisites.first}"
end

file GENERATED_TOKENIZER => "lib/nokogiri/css/tokenizer.rex" do |t|
  sh "frex -i --independent -o #{t.name} #{t.prerequisites.first}"
end

task 'ext/nokogiri/Makefile' do
  Dir.chdir('ext/nokogiri') do
    ruby "extconf.rb"
  end
end

task EXT => 'ext/nokogiri/Makefile' do
  Dir.chdir('ext/nokogiri') do
    sh 'make'
  end
end

task :build => [EXT, GENERATED_PARSER, GENERATED_TOKENIZER]

require 'open-uri'
namespace :build do
  namespace :win32 do
    file 'cross/bin/ruby.exe' => ['cross/ruby-1.8.6-p287'] do
      Dir.chdir('cross/ruby-1.8.6-p287') do
        str = ''
        File.open('Makefile.in', 'rb') do |f|
          f.each_line do |line|
            if line =~ /^\s*ALT_SEPARATOR =/
              str += "\t\t    " + 'ALT_SEPARATOR = "\\\\\"; \\'
              str += "\n"
            else
              str += line
            end
          end
        end
        File.open('Makefile.in', 'wb') { |f| f.write str }
        sh(<<-eocommand)
          env ac_cv_func_getpgrp_void=no \
            ac_cv_func_setpgrp_void=yes \
            rb_cv_negative_time_t=no \
            ac_cv_func_memcmp_working=yes \
            rb_cv_binary_elf=no \
            ./configure \
            --host=i386-mingw32 \
            --target=i386-mingw32 \
            --prefix=#{File.expand_path(File.join(Dir.pwd, '..'))}
        eocommand
        sh 'make'
        sh 'make install'
      end
    end

    desc 'build cross compiled ruby'
    task :ruby => 'cross/bin/ruby.exe'
  end

  desc 'build nokogiri for win32'
  task :win32 => [GENERATED_PARSER, GENERATED_TOKENIZER, 'build:externals', 'build:win32:ruby'] do
    dash_i = File.expand_path(
      File.join(File.dirname(__FILE__), 'cross/lib/ruby/1.8/i386-mingw32/')
    )
    Dir.chdir('ext/nokogiri') do
      ruby " -I #{dash_i} extconf.rb"
      sh 'make'
    end
    dlls = Dir[File.join(File.dirname(__FILE__), 'cross', '**/*.dll')]
    dlls.each do |dll|
      next if dll =~ /ruby/
      cp dll, 'ext/nokogiri'
    end
  end

  libs = %w{
    iconv-1.9.2.win32
    zlib-1.2.3.win32
    libxml2-2.7.1.win32
    libxslt-1.1.24.win32
  }

  libs.each do |lib|
    file "cross/#{lib}" do |t|
      puts "downloading #{lib}"
      FileUtils.mkdir_p('cross')
      Dir.chdir('cross') do
        File.open("#{lib}.zip", 'wb') { |f|
          f.write open("http://www.zlatkovic.com/pub/libxml/#{lib}.zip").read
        }
        sh "unzip #{lib}.zip"
      end
    end
  end

  file 'cross/ruby-1.8.6-p287' do |t|
    puts "downloading ruby"
    FileUtils.mkdir_p('cross')
    Dir.chdir('cross') do
      File.open("ruby-1.8.6-p287.tar.gz", 'wb') { |f|
        f.write open("ftp://ftp.ruby-lang.org/pub/ruby/1.8/ruby-1.8.6-p287.tar.gz").read
      }
      sh "tar zxvf ruby-1.8.6-p287.tar.gz"
    end
  end

  task :externals => libs.map { |x| "cross/#{x}" } + ['cross/ruby-1.8.6-p287']
end

Rake::Task[:test].prerequisites << :build
Rake::Task[:check_manifest].prerequisites << GENERATED_PARSER
Rake::Task[:check_manifest].prerequisites << GENERATED_TOKENIZER

# vim: syntax=Ruby
