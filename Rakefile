require 'rubygems'
require 'rake'
require 'json'

$LOAD_PATH.unshift File.expand_path("../lib", __FILE__)

require 'rake/extensiontask'
Rake::ExtensionTask.new do |ext|
  ext.name = 'c_tokenizer'
  ext.ext_dir = 'ext/jsduck'
  ext.lib_dir = 'lib/jsduck'
end

require 'rspec'
require 'rspec/core/rake_task'
RSpec::Core::RakeTask.new(:spec) do |spec|
  spec.rspec_opts = ["--color"]
  spec.pattern = "spec/**/*_spec.rb"
end

def load_sdk_vars
  if File.exists?("sdk-vars.rb")
    require "sdk-vars.rb"
  else
    puts "Error: sdk-vars.rb not found."
    puts
    puts "Please create file sdk-vars.rb and define constants SDK_DIR and OUT_DIR in it."
    puts
    puts "For example:"
    puts
    puts "    # path to Ext JS 4 build"
    puts "    EXT_DIR='/path/to/ext-4.0.7'"
    puts "    # where to output the docs"
    puts "    OUT_DIR='/path/to/ouput/dir'"
    puts "    # path to SDK (for developers at Sencha)"
    puts "    SDK_DIR='/path/to/SDK'"
    puts "    # path to Animator (for developers at Sencha)"
    puts "    ANIMATOR_DIR='/path/to/Animator'"
    exit 1
  end
end

# Compress JS/CSS file in-place
# Using a hackish way to access yui-compressor
def yui_compress(fname)
  system "java -jar $(dirname $(which sencha))/../jsbuilder/ycompressor/ycompressor.jar -o #{fname} #{fname}"
end

# Reads in all CSS files referenced between BEGIN CSS and END CSS markers.
# Deletes those input CSS files and writes out concatenated CSS to
# resources/css/app.css
# Finally replaces the CSS section with <link> to that one CSS file.
def combine_css(html, dir, opts = :write)
  css_section_re = /<!-- BEGIN CSS -->.*<!-- END CSS -->/m
  css = []
  css_section_re.match(html)[0].each_line do |line|
    if line =~ /<link rel="stylesheet" href="(.*?)"/
      file = $1
      css << IO.read(dir + "/" + file)
      system("rm", dir + "/" + file) if opts == :write
    end
  end

  if opts == :write
    fname = "#{dir}/resources/css/app.css"
    File.open(fname, 'w') {|f| f.write(css.join("\n")) }
    yui_compress(fname)
  end
  html.sub(css_section_re, '<link rel="stylesheet" href="resources/css/app.css" type="text/css" />')
end

# Same thing for JavaScript, result is written to: app.js
def combine_js(html, dir)
  js_section_re = /<!-- BEGIN JS -->.*<!-- END JS -->/m
  js = []
  js_section_re.match(html)[0].each_line do |line|
    if line =~ /<script .* src="(.*)">/
      file = $1
      js << IO.read(dir + "/" + file)
      if file !~ /ext\.js/
        system("rm", dir + "/" + file)
      end
    elsif line =~ /<script .*>(.*)<\/script>/
      js << $1
    end
  end

  fname = "#{dir}/app.js"
  File.open(fname, 'w') {|f| f.write(js.join("\n")) }
  yui_compress(fname)
  html.sub(js_section_re, '<script type="text/javascript" src="app.js"></script>')
end

# Compress JavaScript and CSS files of JSDuck
def compress
  load_sdk_vars

  # Clean up template-min/ left over from previous compress task
  system("rm", "-rf", "template-min")
  # Copy template/ files to template-min/
  system("cp", "-r", "template", "template-min")
  # Now do everything that follows in template-min/ dir
  dir = "template-min"

  # Create JSB3 file for Docs app
  system("sencha", "create", "jsb", "-a", "#{dir}/build-js.html", "-p", "#{dir}/app.jsb3")
  # Concatenate files listed in JSB3 file
  system("sencha", "build", "-p", "#{dir}/app.jsb3", "-d", dir)
  # Remove intermediate build files
  system("rm", "#{dir}/app.jsb3")
  system("rm", "#{dir}/all-classes.js")
  # Replace app.js with app-all.js
  system("mv", "#{dir}/app-all.js", "#{dir}/app.js")
  # Remove the entire app/ dir
  system("rm", "-r", "#{dir}/app")

  # Concatenate CSS in print-template.html file
  print_template = "#{dir}/print-template.html";
  html = IO.read(print_template);
  # Just modify HTML to link app.css, don't write files.
  html = combine_css(html, dir, :replace_html_only)
  File.open(print_template, 'w') {|f| f.write(html) }

  # Concatenate CSS and JS files referenced in template.html file
  template_html = "#{dir}/template.html"
  html = IO.read(template_html)
  html = combine_css(html, dir)
  html = combine_js(html, dir)
  File.open(template_html, 'w') {|f| f.write(html) }

  # Clean up SASS files
  # (But keep prettify lib, which is needed for source files)
  system "rm -rf #{dir}/resources/sass"
  system "rm -rf #{dir}/resources/codemirror"
  system "rm -rf #{dir}/resources/.sass-cache"

  # Empty the extjs dir, leave only the main JS files, CSS and images
  system "rm -rf #{dir}/extjs"
  system "mkdir #{dir}/extjs"
  system "cp #{EXT_DIR}/ext-all.js #{dir}/extjs"
  system "cp #{EXT_DIR}/ext-all-debug.js #{dir}/extjs"
  system "cp #{EXT_DIR}/bootstrap.js #{dir}/extjs"
  system "mkdir -p #{dir}/extjs/resources/css"
  system "cp #{EXT_DIR}/resources/css/ext-all.css #{dir}/extjs/resources/css"
  system "mkdir -p #{dir}/extjs/resources/themes/images"
  system "cp -r #{EXT_DIR}/resources/themes/images/default #{dir}/extjs/resources/themes/images"
end


class JsDuckRunner
  def initialize
    @options = []
    load_sdk_vars
    @sdk_dir = SDK_DIR
    @out_dir = OUT_DIR
    @ext_dir = EXT_DIR
    @animator_dir = ANIMATOR_DIR
  end

  def add_options(options)
    @options += options
  end

  def add_relative_examples_path
    @options += ["--head-html", <<-EOHTML]
      <script type="text/javascript">
        Docs.exampleBaseUrl = "#{relative_sdk_path}examples/";
      </script>
    EOHTML
  end

  # Enables comments when CORS is supported by browser.
  # This excludes Opera and IE < 8.
  # We check explicitly for IE version to make sure the code works the
  # same way in both real IE7 and IE7-mode of IE8/9.
  def add_comments(db_name)
    comments_base_url = "http://projects.sencha.com/auth"
    @options += ["--head-html", <<-EOHTML]
      <script type="text/javascript">
        Docs.enableComments = ("withCredentials" in new XMLHttpRequest()) || (Ext.ieVersion >= 8);
        Docs.baseUrl = "#{comments_base_url}";
        Docs.commentsDb = "#{db_name}";
      </script>
    EOHTML
  end

  # For export of ExtJS, reference extjs from the parent dir
  def make_paths_relative
    relative_sdk_path = "../"
    ["#{@out_dir}/eg-iframe.html", "#{@out_dir}/index.html"].each do |file|
      out = []
      IO.read(file).each_line do |line|
        out << line.sub(/((src|href)="extjs\/)/, '\2="' + relative_sdk_path)
      end
      File.open(file, 'w') {|f| f.write(out) }
    end
  end

  def add_ext4
    @options += [
      "--title", "Sencha Docs - Ext JS 4.0",
      "--footer", "Ext JS 4.0 Docs - Generated with <a href='https://github.com/senchalabs/jsduck'>JSDuck</a> VERSION. <a href='http://www.sencha.com/legal/terms-of-use/'>Terms of Use</a>",
      "--ignore-global",
      "--no-warnings",
      "--images", "#{@ext_dir}/docs/doc-resources",
      "--local-storage-db", "ext-4",
      "--output", "#{@out_dir}",
      "#{@ext_dir}/src",
    ]
  end

  def add_touch_export
    @options += [
      "--json",
      "--output", "#{@out_dir}/../export/touch1",
      "--external=google.maps.Map,google.maps.LatLng",
      "#{@sdk_dir}/touch/sencha-touch.jsb3",
    ]
  end

  def add_touch2_export
    @options += [
      "--export", "full",
      "--output", "#{@out_dir}/../export/touch2",
      "--external=google.maps.Map,google.maps.LatLng",
      "#{@sdk_dir}/touch/touch.jsb3",
    ]
  end

  def set_touch2_src
    relative_touch_path = "../"
    system("cp", "-r", "#{@sdk_dir}/touch/docs/welcome.html", "template-min/welcome.html")
    system("cp", "-r", "#{@sdk_dir}/touch/docs/eg-iframe.html", "template-min/eg-iframe.html")

    ["template-min/eg-iframe.html", "template-min/welcome.html"].each do |file|
      html = IO.read(file);

      touch_src_re = /((src|href)="touch)/m
      out = []

      html.each_line do |line|
        out << line.sub(/((src|href)="touch\/)/, '\2="' + relative_touch_path)
      end

      File.open(file, 'w') {|f| f.write(out) }
    end

    head_html = <<-EOHTML
      <script type="text/javascript">
        Docs.exampleBaseUrl = "#{relative_touch_path}examples/";
        if (Ext.is.Phone) { window.location = "#{relative_touch_path}examples/"; }
      </script>
    EOHTML

    @options += [
      "--body-html", head_html,
      "--welcome", "template-min/welcome.html",
      "--eg-iframe", "template-min/eg-iframe.html"
    ]
  end

  def add_debug
    @options += [
      "--extjs-path", "extjs/ext-all-debug.js",
      "--template-links",
      "--template", "template",
    ]
  end

  def add_seo
    @options += [
      "--seo",
    ]
  end

  def add_export_notice path
    @options += [
      "--body-html", <<-EOHTML
      <div id="notice-text" style="display: none">
        Use <a href="http://docs.sencha.com/#{path}">http://docs.sencha.com/#{path}</a> for up to date documentation and features
      </div>
      EOHTML
    ]
  end

  def add_google_analytics
    @options += [
      "--body-html", <<-EOHTML
      <script type="text/javascript">
        var _gaq = _gaq || [];
        _gaq.push(['_setAccount', 'UA-1396058-10']);
        _gaq.push(['_trackPageview']);
        (function() {
          var ga = document.createElement('script');
          ga.type = 'text/javascript';
          ga.async = true;
          ga.src = ('https:' == document.location.protocol ? 'https://ssl' : 'http://www') + '.google-analytics.com/ga.js';
          var s = document.getElementsByTagName('script')[0];
          s.parentNode.insertBefore(ga, s);
        })();

        Docs.initEventTracking = function() {
            Docs.App.getController('Classes').addListener({
                showClass: function(cls) {
                    _gaq.push(['_trackEvent', 'Classes', 'Show', cls]);
                },
                showMember: function(cls, anchor) {
                    _gaq.push(['_trackEvent', 'Classes', 'Member', cls + ' - ' + anchor]);
                }
            });
            Docs.App.getController('Guides').addListener({
                showGuide: function(guide) {
                    _gaq.push(['_trackEvent', 'Guides', 'Show', guide]);
                }
            });
            Docs.App.getController('Videos').addListener({
                showVideo: function(video) {
                    _gaq.push(['_trackEvent', 'Video', 'Show', video]);
                }
            });
            Docs.App.getController('Examples').addListener({
                showExample: function(example) {
                    _gaq.push(['_trackEvent', 'Example', 'Show', example]);
                }
            });
        }

        Docs.otherProducts = [
          {
              text: 'Ext JS 4',
              href: 'http://docs.sencha.com/ext-js/4-0'
          },
          {
              text: 'Ext JS 3',
              href: 'http://docs.sencha.com/ext-js/3-4'
          },
          {
              text: 'Sencha Touch 2',
              href: 'http://docs.sencha.com/touch/2-0'
          },
          {
              text: 'Sencha Touch 1',
              href: 'http://docs.sencha.com/touch/1-1'
          },
          {
              text: 'Touch Charts',
              href: 'http://docs.sencha.com/touch-charts/1-0'
          },
          {
              text: 'Sencha Animator',
              href: 'http://docs.sencha.com/animator/1-0'
          },
          {
              text: 'Sencha.io',
              href: 'http://docs.sencha.com/io/1-0'
          }
        ];
      </script>
    EOHTML
    ]

  end

  # Copy over SDK examples
  def copy_sdk_examples
    system "mkdir #{@out_dir}/extjs/builds"
    system "cp #{@ext_dir}/builds/ext-core.js #{@out_dir}/extjs/builds/ext-core.js"
    system "cp #{@ext_dir}/release-notes.html #{@out_dir}/extjs"
    system "cp -r #{@ext_dir}/examples #{@out_dir}/extjs"
    system "cp -r #{@ext_dir}/welcome #{@out_dir}/extjs"
  end

  # Copy over Sencha Touch
  def copy_touch2_build
    system "cp -r #{@sdk_dir}/touch/build #{@out_dir}/touch"
  end

  def run
    # Pass multiple arguments to system, so we'll take advantage of the built-in escaping
    system(*["ruby", "bin/jsduck"].concat(@options))
  end
end

# Run compass to generate CSS files
task :sass do
  system "compass compile --quiet template/resources/sass"
end

desc "Run JSDuck on Ext JS SDK (for internal use at Sencha)\n" +
     "sdk         - creates debug/development version\n" +
     "sdk[export] - creates export version\n" +
     "sdk[live]   - create live version for deployment\n"
task :sdk, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export", "live"].include?(mode)
  compress if mode == "export" || mode == "live"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/extjs/docs/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_export_notice("ext-js/4-0") if mode == "export"
  runner.add_google_analytics if mode == "live"
  runner.add_comments('comments-ext-js-4') if mode == "debug" || mode == "live"
  runner.run

  runner.copy_sdk_examples if mode == "export" || mode == "live"
  runner.make_paths_relative if mode == "export"
end

desc "Run JSDuck on Docs app itself"
task :docs do
  runner = JsDuckRunner.new
  runner.add_ext4
  runner.add_options([
    "--builtin-classes",
    "template/app"
  ])
  runner.add_debug
  runner.add_seo
  runner.run
end

desc "Run JSDuck on official Ext JS 4.0.2a build\n" +
     "ext4             - creates debug/development version\n" +
     "ext4[export]     - creates export/deployable version\n"
task :ext4, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export"].include?(mode)
  compress if mode == "export"

  runner = JsDuckRunner.new
  runner.add_ext4
  runner.add_debug if mode == "debug"
  runner.add_seo
  runner.run
end

desc "Run JSDuck on official Ext JS 3.4 build\n" +
     "ext3             - creates debug/development version\n" +
     "ext3[export]     - creates export/deployable version\n"
     "ext3[live]       - creates live version for deployment\n"
task :ext3, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export", "live"].include?(mode)
  compress if mode == "export"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/../ext-3.4.0/src/doc-config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.run
end

desc "Run JSDuck on Sencha Touch (for internal use at Sencha)\n" +
     "touch       - creates debug/development version\n" +
     "touch[live] - create live version for deployment\n"
task :touch, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "live"].include?(mode)
  compress if mode == "live"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/touch/doc-resources/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.run
end

desc "Run JSDuck on Sencha Touch 2 (for internal use at Sencha)\n" +
     "touch2         - creates debug/development version\n" +
     "touch2[export] - creates export version\n" +
     "touch2[live]   - create live version for deployment\n"
task :touch2, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export", "live"].include?(mode)
  compress if mode == "live" || mode == "export"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/touch/docs/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_export_notice("touch/2-0") if mode == "export"
  runner.set_touch2_src if mode == "export"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.add_comments('comments-touch-2') if mode == "debug" || mode == "live"
  runner.run

  runner.copy_touch2_build if mode != "export"
end

desc "Run JSDuck on Sencha Touch Charts (for internal use at Sencha)\n" +
     "charts         - creates debug/development version\n" +
     "charts[export] - create live version for deployment\n"
     "charts[live]   - create live version for deployment\n"
task :charts, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export", "live"].include?(mode)
  compress if mode == "live"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/charts/docs/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.run
end

desc "Run JSDuck on Sencha.IO Sync (for internal use at Sencha)\n" +
     "senchaio         - creates debug/development version\n" +
     "senchaio[export] - create live version for deployment\n"
     "senchaio[live]   - create live version for deployment\n"
task :senchaio, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "export", "live"].include?(mode)
  compress if mode == "live"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{SDK_DIR}/../sync/docs/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.run
end

desc "Run JSDuck on Sencha Animator (for internal use at Sencha)\n" +
     "animator         - creates debug/development version\n" +
     "animator[export] - create live version for deployment\n"
     "animator[live]   - create live version for deployment\n"
task :animator, [:mode] => :sass do |t, args|
  mode = args[:mode] || "debug"
  throw "Unknown mode #{mode}" unless ["debug", "live", "export"].include?(mode)
  compress if mode == "live"

  runner = JsDuckRunner.new
  runner.add_options ["--output", OUT_DIR, "--config", "#{ANIMATOR_DIR}/docs/config.json"]
  runner.add_debug if mode == "debug"
  runner.add_seo if mode == "debug" || mode == "live"
  runner.add_google_analytics if mode == "live"
  runner.run
end

desc "Run JSDuck JSON Export (for internal use at Sencha)\n" +
     "export[touch]  - creates export for Touch 1\n" +
     "export[touch2] - creates export for Touch 2"
task :export, [:mode] do |t, args|
  mode = args[:mode]
  throw "Unknown mode #{mode}" unless ["touch", "touch2"].include?(mode)

  runner = JsDuckRunner.new
  runner.add_touch_export if mode == "touch"
  runner.add_touch2_export if mode == "touch2"
  runner.run
end

desc "Build JSDuck gem"
task :gem => :sass do
  compress
  system "gem build jsduck.gemspec"
end

task :default => :spec
