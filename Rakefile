# -*- ruby -*-

require "bundler/gem_tasks"

require 'rdoc/task'

spec = Gem::Specification.load("racc.gemspec")

RDoc::Task.new(:docs) do |rd|
  rd.main = "README.en.rdoc"
  rd.rdoc_files.include(spec.files.find_all { |file_name|
    file_name =~ /^(bin|lib|ext)/ || file_name !~ /\//
  })

  title = "#{spec.name}-#{spec.version} Documentation"

  rd.options << "-t #{title}"
end

def java?
  /java/ === RUBY_PLATFORM
end
def jruby?
  Object.const_defined?(:RUBY_ENGINE) and 'jruby' == RUBY_ENGINE
end

file 'lib/racc/parser-text.rb' => ['lib/racc/parser.rb', 'lib/racc/info.rb', __FILE__] do |t|
  source = 'lib/racc/parser.rb'

  text = File.read(source)
  text.sub!(/\A# *frozen[-_]string[-_]literal:.*\n/, '')
  text.sub!(/\A(?:#.*\n)*+(?:(?:[^#\n].*)?\n)*+\K(?:#.*\n)++/, '')
  libs = []
  text.gsub!(/(\A|\n)require '(.*)'\n/) do
    pre, lib = $1, $2
    code = File.read("lib/#{lib}.rb")
    code.sub!(/\A(?:#.*\n)+/, '')
    if code.sub!(/\A\s*^module Racc\n((?m:.*)\n)end\Z/, '\1')
      code.sub!(/\n\K\n+\z/, '')
      libs << code
      ''
    else
      pre + <<-CODE
unless $".find {|p| p.end_with?('/#{lib}.rb')}
$".push "\#{__dir__}/#{lib}.rb"
#{code}
end
      CODE
    end
  rescue
    $&
  end
  unless libs.empty?
    text.sub!(/^module Racc\n\K/, libs.join("\n"))
  end
  text.sub!(/^(?=module Racc$)/, "#--\n")
  File.open(t.name, 'wb') { |io|
    io.write(<<-eorb)
module Racc
  PARSER_TEXT = <<'__end_of_file__'
#{text}
\#++
__end_of_file__
end
    eorb
  }
end

javasrc, = Dir.glob('ext/racc/**/Cparse.java')
task :compile => javasrc do
  code = File.binread(javasrc)
  if code.sub!(/RACC_VERSION\s*=\s*"\K([^"]*)(?=")/) {|v| break if v == spec.version; spec.version}
    File.binwrite(javasrc, code)
  end
end

lib_dir = nil # for dummy rake/extensiontask.rb at ruby test-bundled-gems
if jruby?
  # JRUBY
  require "rake/javaextensiontask"
  extask = Rake::JavaExtensionTask.new("cparse") do |ext|
    jruby_home = RbConfig::CONFIG['prefix']
    lib_dir = (ext.lib_dir << "/#{ext.platform}/racc")
    ext.ext_dir = 'ext/racc'
    # source/target jvm
    ext.source_version = '1.8'
    ext.target_version = '1.8'
    jars = ["#{jruby_home}/lib/jruby.jar"] + FileList['lib/*.jar']
    ext.classpath = jars.map { |x| File.expand_path x }.join( ':' )
    ext.name = 'cparse-jruby'
  end

  task :build => "#{extask.lib_dir}/#{extask.name}.jar"
else
  # MRI
  require "rake/extensiontask"
  extask = Rake::ExtensionTask.new "cparse" do |ext|
    lib_dir = (ext.lib_dir << "/#{RUBY_VERSION}/#{ext.platform}/racc")
    ext.ext_dir = 'ext/racc/cparse'
  end
end

desc 'Make autogenerated sources'
task :srcs => ['lib/racc/parser-text.rb']

task :compile => :srcs
task :build => :srcs

task :test => :compile

require 'rake/testtask'

Rake::TestTask.new(:test) do |t|
  t.libs << lib_dir if lib_dir
  t.libs << "test/lib"
  ENV["RUBYOPT"] = "-I" + t.libs.join(File::PATH_SEPARATOR)
  t.ruby_opts << "-rhelper"
  t.test_files = FileList["test/**/test_*.rb"]
  if RUBY_VERSION >= "2.6"
    t.ruby_opts << "--enable-frozen-string-literal"
    t.ruby_opts << "--debug=frozen-string-literal" if RUBY_ENGINE != "truffleruby"
  end
end
