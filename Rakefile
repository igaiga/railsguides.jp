namespace :guides do
  desc 'Generate guides (for authors), use ONLY=foo to process just "foo.md"'
  task generate: "generate:html"

  # Guides are written in UTF-8, but the environment may be configured for some
  # other locale, these tasks are responsible for ensuring the default external
  # encoding is UTF-8.
  #
  # Real use cases: Generation was reported to fail on a machine configured with
  # GBK (Chinese). The docs server once got misconfigured somehow and had "C",
  # which broke generation too.
  task :encoding do
    %w(LANG LANGUAGE LC_ALL).each do |env_var|
      ENV[env_var] = "en_US.UTF-8"
    end
  end

  namespace :generate do
    desc "Generate HTML guides"
    task html: :encoding do
      ENV["WARNINGS"] = "1" # authors can't disable this
      ruby "rails_guides.rb"
    end

    desc "Generate .mobi file. The kindlegen executable must be in your PATH. You can get it for free from http://www.amazon.com/gp/feature.html?docId=1000765211"
    task kindle: :encoding do
      require "kindlerb"
      unless Kindlerb.kindlegen_available?
        abort "Please run `setupkindlerb` to install kindlegen"
      end
      unless `convert` =~ /convert/
        abort "Please install ImageMagick"
      end
      ENV["KINDLE"] = "1"
      Rake::Task["guides:generate:html"].invoke
    end
  end

  # Validate guides -------------------------------------------------------------------------
  desc 'Validate guides, use ONLY=foo to process just "foo.html"'
  task validate: :encoding do
    ruby "w3c_validator.rb"
  end

  desc "Show help"
  task :help do
    puts <<HELP

Guides are taken from the source directory, and the result goes into the
output directory. Assets are stored under files, and copied to output/files as
part of the generation process.

You can generate HTML, Kindle or both formats using the `guides:generate` task.

All of these processes are handled via rake tasks, here's a full list of them:

#{%x[rake -T]}
Some arguments may be passed via environment variables:

  RAILS_VERSION=tag
    If guides are being generated for a specific Rails version set the Git tag
    here, otherwise the current SHA1 is going to be used to generate edge guides.

  ALL=1
    Force generation of all guides.

  ONLY=name
    Useful if you want to generate only one or a set of guides.

    Generate only association_basics.html:
      ONLY=assoc

    Separate many using commas:
      ONLY=assoc,migrations

  GUIDES_LANGUAGE
    Use it when you want to generate translated guides in
    source/<GUIDES_LANGUAGE> folder (such as source/es)

Examples:
  $ rake guides:generate ALL=1 RAILS_VERSION=v5.1.0
  $ rake guides:generate ONLY=migrations
  $ rake guides:generate:kindle
  $ rake guides:generate GUIDES_LANGUAGE=es
HELP
  end
end

task default: "guides:help"
