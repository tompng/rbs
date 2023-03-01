require "bundler/gem_tasks"
require "rake/testtask"
require "rbconfig"
require 'rake/extensiontask'

$LOAD_PATH << File.join(__dir__, "test")

ruby = ENV["RUBY"] || RbConfig.ruby
rbs = File.join(__dir__, "exe/rbs")
bin = File.join(__dir__, "bin")

Rake::ExtensionTask.new("rbs_extension")

Rake::TestTask.new(:test => :compile) do |t|
  t.libs << "test"
  t.libs << "lib"
  t.test_files = FileList["test/**/*_test.rb"].reject do |path|
    path =~ %r{test/stdlib/}
  end
end

multitask :default => [:test, :stdlib_test, :rubocop, :validate, :test_doc]

task :lexer do
  sh "re2c -W --no-generation-date -o ext/rbs_extension/lexer.c ext/rbs_extension/lexer.re"
end

task :confirm_lexer => :lexer do
  puts "Testing if lexer.c is updated with respect to lexer.re"
  sh "git diff --exit-code ext/rbs_extension/lexer.c"
end

rule ".c" => ".re" do |t|
  puts "⚠️⚠️⚠️ #{t.name} is older than #{t.source}. You may need to run `rake lexer` ⚠️⚠️⚠️"
end

task :annotate do
  sh "bin/generate_docs.sh"
end

task :confirm_annotation do
  puts "Testing if RBS docs are updated with respect to RDoc"
  sh "git diff --exit-code core stdlib"
end

task :compile => "ext/rbs_extension/lexer.c"

task :test_doc do
  files = Dir.chdir(File.expand_path('..', __FILE__)) do
    `git ls-files -z`.split("\x0").select do |file| Pathname(file).extname == ".md" end
  end

  sh "#{ruby} #{__dir__}/bin/run_in_md.rb #{files.join(" ")}"
end

task :validate => :compile do
  require 'yaml'

  sh "#{ruby} #{rbs} validate --silent"

  libs = FileList["stdlib/*"].map {|path| File.basename(path).to_s }
  libs << "rbs"

  libs.each do |lib|
    sh "#{ruby} #{rbs} -r #{lib} validate --silent"
  end
end

FileList["test/stdlib/**/*_test.rb"].each do |test|
  task test => :compile do
    sh "#{ruby} -Ilib #{bin}/test_runner.rb #{test}"
  end
end

task :stdlib_test do
  test_files = FileList["test/stdlib/**/*_test.rb"].reject do |path|
    path =~ %r{Ractor}
  end
  sh "#{ruby} -Ilib #{bin}/test_runner.rb #{test_files.join(' ')}"
  # TODO: Ractor tests need to be run in a separate process
  sh "#{ruby} -Ilib #{bin}/test_runner.rb test/stdlib/Ractor_test.rb"
end

task :rubocop do
  sh "rubocop --parallel"
end

namespace :generate do
  desc "Generate a test file for a stdlib class signatures"
  task :stdlib_test, [:class] do |_task, args|
    klass = args.fetch(:class) do
      raise "Class name is necessary. e.g. rake 'generate:stdlib_test[String]'"
    end

    require "erb"
    require "rbs"

    class TestTarget
      def initialize(klass)
        @type_name = RBS::Namespace.parse(klass).to_type_name
      end

      def path
        Pathname(ENV['RBS_GENERATE_TEST_PATH'] || "test/stdlib/#{file_name}_test.rb")
      end

      def file_name
        @type_name.to_s.gsub(/\A::/, '').gsub(/::/, '_')
      end

      def to_s
        @type_name.to_s
      end

      def absolute_type_name
        @absolute_type_name ||= @type_name.absolute!
      end
    end

    target = TestTarget.new(klass)
    path = target.path
    raise "#{path} already exists!" if path.exist?

    class TestTemplateBuilder
      attr_reader :target, :env

      def initialize(target)
        @target = target

        loader = RBS::EnvironmentLoader.new
        Dir['stdlib/*'].each do |lib|
          next if lib.end_with?('builtin')

          loader.add(library: File.basename(lib))
        end
        @env = RBS::Environment.from_loader(loader).resolve_type_names
      end

      def call
        ERB.new(<<~ERB, trim_mode: "-").result(binding)
          require_relative "test_helper"

          <%- unless class_methods.empty? -%>
          class <%= target %>SingletonTest < Test::Unit::TestCase
            include TypeAssertions

            # library "pathname", "securerandom"     # Declare library signatures to load
            testing "singleton(::<%= target %>)"

          <%- class_methods.each do |method_name, definition| %>
            def test_<%= test_name_for(method_name) %>
          <%- definition.method_types.each do |method_type| -%>
              assert_send_type  "<%= method_type %>",
                                <%= target %>, :<%= method_name %>
          <%- end -%>
            end
          <%- end -%>
          end
          <%- end -%>

          <%- unless instance_methods.empty? -%>
          class <%= target %>Test < Test::Unit::TestCase
            include TypeAssertions

            # library "pathname", "securerandom"     # Declare library signatures to load
            testing "::<%= target %>"

          <%- instance_methods.each do |method_name, definition| %>
            def test_<%= test_name_for(method_name) %>
          <%- definition.method_types.each do |method_type| -%>
              assert_send_type  "<%= method_type %>",
                                <%= target %>.new, :<%= method_name %>
          <%- end -%>
            end
          <%- end -%>
          end
          <%- end -%>
        ERB
      end

      private

      def test_name_for(method_name)
        {
          :==  => 'double_equal',
          :!=  => 'not_equal',
          :=== => 'triple_equal',
          :[]  => 'square_bracket',
          :[]= => 'square_bracket_assign',
          :>   => 'greater_than',
          :<   => 'less_than',
          :>=  => 'greater_than_equal_to',
          :<=  => 'less_than_equal_to',
          :<=> => 'spaceship',
          :+   => 'plus',
          :-   => 'minus',
          :*   => 'multiply',
          :/   => 'divide',
          :**  => 'power',
          :%   => 'modulus',
          :&   => 'and',
          :|   => 'or',
          :^   => 'xor',
          :>>  => 'right_shift',
          :<<  => 'left_shift',
          :=~  => 'pattern_match',
          :!~  => 'does_not_match',
          :~   => 'tilde'
        }.fetch(method_name, method_name)
      end

      def class_methods
        @class_methods ||= RBS::DefinitionBuilder.new(env: env).build_singleton(target.absolute_type_name).methods.select {|_, definition|
          definition.implemented_in == target.absolute_type_name
        }
      end

      def instance_methods
        @instance_methods ||= RBS::DefinitionBuilder.new(env: env).build_instance(target.absolute_type_name).methods.select {|_, definition|
          definition.implemented_in == target.absolute_type_name
        }
      end
    end

    path.write TestTemplateBuilder.new(target).call

    puts "Created: #{path}"
  end
end

task :test_generate_stdlib do
  sh "RBS_GENERATE_TEST_PATH=/tmp/Array_test.rb rake 'generate:stdlib_test[Array]'"
  sh "ruby -c /tmp/Array_test.rb"
  sh "RBS_GENERATE_TEST_PATH=/tmp/Thread_Mutex_test.rb rake 'generate:stdlib_test[Thread::Mutex]'"
  sh "ruby -c /tmp/Thread_Mutex_test.rb"
end

Rake::Task[:release].enhance do
  Rake::Task[:"release:note"].invoke
end

namespace :release do
  desc "Explain the post-release steps automatically"
  task :note do
    version = Gem::Version.new(RBS::VERSION)
    major, minor, patch, *_ = RBS::VERSION.split(".")
    major = major.to_i
    minor = minor.to_i
    patch = patch.to_i

    puts "🎉🎉🎉🎉 Congratulations for **#{version}** release! 🎉🎉🎉🎉"
    puts
    puts "There are a few things left to complete the release. 💪"
    puts

    if patch == 0 || version.prerelease?
      puts "* [ ] Update release note: https://github.com/ruby/rbs/wiki/Release-Note-#{major}.#{minor}"
    end

    if patch == 0 && !version.prerelease?
      puts "* [ ] Delete `RBS XYZ is the latest version of...` from release note: https://github.com/ruby/rbs/wiki/Release-Note-#{major}.#{minor}"
    end

    puts "* [ ] Publish a release at GitHub"
    puts "* [ ] Make some announcements on Twitter/Mustdon/Slack/???"

    puts
    puts

    puts "✏️ Making a draft release on GitHub..."

    content = File.read(File.join(__dir__, "CHANGELOG.md"))
    changelog = content.scan(/^## \d.*?(?=^## \d)/m)[0]
    changelog = changelog.sub(/^.*\n^.*\n/, "").rstrip

    notes = <<NOTES
[Release note](https://github.com/ruby/rbs/wiki/Release-Note-#{major}.#{minor})

#{changelog}
NOTES

    command = [
      "gh",
      "release",
      "create",
      "--draft",
      "v#{RBS::VERSION}",
      "--title=#{RBS::VERSION}",
      "--notes=#{notes}"
    ]

    if version.prerelease?
      command << "--prerelease"
    end

    require "open3"
    output, status = Open3.capture2(*command)
    if status.success?
      puts "  >> Done! Open #{output.chomp} and publish the release!"
    end
  end
end
