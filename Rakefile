# macOS release pipeline for Tuist + Sparkle + GitHub.
# Copy this file to your project's root. Requires Ruby 2.6+, Rake, REXML,
# Tuist, Xcode command-line tools, and an authenticated GitHub CLI (gh).
#
# Configure with environment variables or a project-local release.json:
# {
#   "SCHEME": "MyApp",
#   "TEAM_ID": "YOUR_TEAM_ID",
#   "NOTARY_PROFILE": "my-notary-profile",
#   "SPARKLE_ACCOUNT": "com.example.MyApp"
# }
# Environment variables take precedence. release:setup can add /release.json to .gitignore.
# Credentials and the Sparkle private key stay in Keychain.
# APPLE_ID is prompted for only when creating new notarization credentials.
#
# Optional settings:
# WORKSPACE       Path relative to this file; otherwise the sole *.xcworkspace.
# GH_REPO         owner/repo; otherwise inferred from a github.com origin URL.
# APP_NAME        Exported bundle name without .app; otherwise the sole .app.
# CONFIGURATION   Release (default).
# BUILD_DIR       build (default); reuse a directory to resume a release.
# MANIFEST        Project.swift (default), used only by release:bump.
# SPARKLE_BIN_DIR Tuist/.build/artifacts/sparkle/Sparkle/bin (default).
# RELEASE_NOTES   Markdown file relative to this file; otherwise generic notes.
#
# rake release:setup (create or complete release.json and configure notarization)
# rake release:bump VERSION=1.1 (requires gh access to check published build numbers)
# rake release:meta:diff (compare this Rakefile with the upstream main branch)
# rake release:meta:upgrade (replace this Rakefile with the upstream main branch)
# rake release
# rake -T
#
# Stages can also run independently: release:run:preflight, release:run:archive,
# release:run:export, release:run:zip, release:run:notarize,
# release:run:appcast, release:run:github.
# This workflow builds a universal macOS app with automatic Developer ID signing,
# notarizes a ZIP, and publishes a regular GitHub release marked latest.
# The app's SUFeedURL must be:
# https://github.com/<owner>/<repo>/releases/latest/download/appcast.xml
# SUPublicEDKey must match the Sparkle key stored under SPARKLE_ACCOUNT.
# Commit and push source changes and the v<marketing-version> tag before publishing.
# This pipeline does not commit or push automatically.
# release:bump supports literal string MARKETING_VERSION and numeric string
# CURRENT_PROJECT_VERSION entries in the Tuist manifest; all matches are updated.

require "shellwords"
require "fileutils"
require "json"
require "digest"
require "rexml/document"
require "pathname"
require "net/http"
require "tempfile"
require "rbconfig"
require "open3"

UPSTREAM_RAKEFILE = "https://raw.githubusercontent.com/zachahn/release-tuist-sparkle/main/Rakefile"

class ReleaseSettings
  attr_reader :config, :path

  def initialize(path)
    @path = File.expand_path(path, __dir__)
    @root = File.dirname(@path)
    @config = File.exist?(@path) ? JSON.parse(File.read(@path)) : {}
    abort("release.json must contain a JSON object") unless @config.is_a?(Hash)
  rescue JSON::ParserError => error
    abort("invalid release.json: #{error.message}")
  end

  def build_dir
    @build_dir ||= File.expand_path(setting("BUILD_DIR", "build"), @root)
  end

  def manifest
    File.expand_path(setting("MANIFEST", "Project.swift"), @root)
  end

  def workspace
    configured = optional_setting("WORKSPACE")
    return File.expand_path(configured, @root) if configured
    candidates = Dir.glob(File.join(@root, "*.xcworkspace")).select { |path| File.directory?(path) }
    halt("set WORKSPACE — expected one .xcworkspace in #{@root}, found #{candidates.length}") unless candidates.one?
    candidates.first
  end

  def configured_app_name
    optional_setting("APP_NAME")
  end

  def gh_repo
    @gh_repo ||= begin
      repo = optional_setting("GH_REPO")
      unless repo
        remote = IO.popen(["git", "-C", @root, "remote", "get-url", "origin"], &:read).strip
        halt("command failed: git") unless $?.success?
        match = remote.match(%r{\A(?:https://github\.com/|git@github\.com:|ssh://git@github\.com/)([^/]+/[^/]+?)(?:\.git)?/?\z})
        halt("cannot infer GitHub repository from origin — set GH_REPO=owner/repo") unless match
        repo = match[1]
      end
      halt("GH_REPO must be owner/repo on github.com") unless repo.match?(%r{\A[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+\z})
      repo
    end
  end

  def sparkle_bin_dir
    File.expand_path(setting("SPARKLE_BIN_DIR", "Tuist/.build/artifacts/sparkle/Sparkle/bin"), @root)
  end

  def generate_appcast_bin
    path = File.join(sparkle_bin_dir, "generate_appcast")
    halt("generate_appcast not found — run `tuist install` first") unless File.exist?(path)
    path
  end

  def notary_profile
    setting("NOTARY_PROFILE")
  end

  def scheme
    setting("SCHEME")
  end

  def team_id
    setting("TEAM_ID")
  end

  def sparkle_account
    setting("SPARKLE_ACCOUNT")
  end

  def configuration
    setting("CONFIGURATION", "Release")
  end

  def release_notes
    configured = optional_setting("RELEASE_NOTES")
    File.expand_path(configured, @root) if configured
  end

  private

  def optional_setting(key)
    value = ENV.fetch(key) { config[key] }
    return nil if value.nil?
    abort("#{key} must be a nonempty string") unless value.is_a?(String) && !value.strip.empty?
    value
  end

  # Resolve required settings only when a task needs them, so rake -T works
  # before release credentials have been configured.
  def setting(key, default = nil)
    optional_setting(key) || default ||
      abort("missing #{key} — set it in release.json or pass #{key}=... on the command line")
  end
end

class ReleaseArtifacts
  attr_reader :build_dir, :archive, :export_dir, :dist_dir, :appcast, :source_commit_path

  def initialize(settings)
    @settings = settings
    @build_dir = settings.build_dir
    @archive = File.join(build_dir, "release.xcarchive")
    @export_dir = File.join(build_dir, "export")
    @dist_dir = File.join(build_dir, "dist")
    @appcast = File.join(dist_dir, "appcast.xml")
    @source_commit_path = File.join(build_dir, "release-source-commit")
  end

  def app
    configured = @settings.configured_app_name
    if configured
      halt("APP_NAME must be a bundle name without .app or path separators") if configured.end_with?(".app") || configured.match?(/[\\\/]/) || %w[. ..].include?(configured)
      path = File.join(export_dir, "#{configured}.app")
      halt("missing #{path} — run `rake release:run:export` first") unless File.directory?(path)
      return path
    end
    candidates = Dir.glob(File.join(export_dir, "*.app")).select { |path| File.directory?(path) }
    halt("expected one exported .app, found #{candidates.length} — run `rake release:run:export` or set APP_NAME") unless candidates.one?
    candidates.first
  end

  def app_name
    File.basename(app, ".app")
  end

  # Read a build-setting value from the exported .app's Info.plist.
  def plist(key)
    capture!("/usr/libexec/PlistBuddy", "-c", "Print :#{key}", File.join(app, "Contents", "Info.plist")).strip
  end

  def marketing_version
    plist("CFBundleShortVersionString") # e.g. 1.1
  end

  def build_version
    plist("CFBundleVersion") # e.g. 6 (Sparkle's sparkle:version)
  end

  def tag
    "v#{marketing_version}"
  end

  # GitHub asset names and appcast URLs need a filename without spaces or URL delimiters.
  def zip_name
    "#{app_name.gsub(/[^A-Za-z0-9._-]+/, "-")}-#{marketing_version}-#{build_version}.zip"
  end

  def zip_path
    File.join(dist_dir, zip_name)
  end

  def checksum_path
    "#{zip_path}.sha256"
  end

  def download_prefix
    "https://github.com/#{@settings.gh_repo}/releases/download/#{tag}/"
  end

  def appcast_enclosure
    document = REXML::Document.new(File.read(appcast))
    item = REXML::XPath.match(document, "/rss/channel/item").find do |entry|
      entry.elements["sparkle:version"]&.text == build_version
    end
    enclosure = item&.elements&.[]("enclosure")
    halt("Appcast does not describe this ZIP") unless enclosure &&
      enclosure.attributes["url"] == "#{download_prefix}#{zip_name}" &&
      enclosure.attributes["length"].to_i == File.size(zip_path)
    enclosure
  end
end

class GitHubRelease
  def initialize(settings, artifacts)
    @settings = settings
    @artifacts = artifacts
  end

  def releases
    JSON.parse(capture!("gh", "api", "--paginate", "repos/#{@settings.gh_repo}/releases", "--slurp")).flatten
  end

  def highest_published_build(release_list, excluding_tag = nil)
    release_list.reject { |release| release["draft"] || release["tag_name"] == excluding_tag }
      .flat_map { |release| release.fetch("assets", []) }
      .map { |asset| asset["name"]&.match(/-(\d+)\.zip\z/)&.captures&.first&.to_i }
      .compact
      .max || 0
  end

  def remote_tag_commit(tag)
    reference = JSON.parse(capture!("gh", "api", "repos/#{@settings.gh_repo}/git/ref/tags/#{tag}"))
    object = reference.fetch("object")
    while object["type"] == "tag"
      object = JSON.parse(capture!("gh", "api", "repos/#{@settings.gh_repo}/git/tags/#{object.fetch("sha")}")).fetch("object")
    end
    halt("remote tag #{tag} does not resolve to a commit") unless object["type"] == "commit"
    object.fetch("sha")
  end

  def publish!(source)
    zip_path = @artifacts.zip_path
    halt("missing ZIP or appcast — run earlier steps first") unless File.exist?(zip_path) && File.exist?(@artifacts.appcast)
    source.verify!

    build_version = @artifacts.build_version
    tag = @artifacts.tag
    halt("build #{build_version} is not numeric; expected a monotonically increasing build number") unless build_version.match?(/\A\d+\z/)
    release_list = releases
    prior_build = highest_published_build(release_list, tag)
    halt("build #{build_version} is not newer than published build #{prior_build}") unless build_version.to_i > prior_build
    enclosure = @artifacts.appcast_enclosure
    sh! File.join(@settings.sparkle_bin_dir, "sign_update"), "--account", @settings.sparkle_account,
      "--verify", zip_path, enclosure.attributes["sparkle:edSignature"]
    Dir.chdir(@artifacts.dist_dir) { sh! "shasum", "-a", "256", "-c", File.basename(@artifacts.checksum_path) }

    # Stage assets in a draft so the feed becomes public only after upload succeeds.
    # Existing public assets must stay immutable; reruns may replace draft assets.
    existing = release_list.find { |release| release["tag_name"] == tag }
    assets = [zip_path, @artifacts.checksum_path, @artifacts.appcast]
    if existing
      halt("#{tag} is already published; bump the version first") unless existing["draft"]
      sh! "gh", "release", "upload", tag, *assets, "--repo", @settings.gh_repo, "--clobber"
    else
      supplied_notes = @settings.release_notes
      notes = supplied_notes || File.join(@artifacts.build_dir, "release-notes.md")
      if supplied_notes
        halt("missing RELEASE_NOTES file: #{notes}") unless File.file?(notes)
      else
        File.write(notes, <<~NOTES)
          #{@artifacts.app_name} #{@artifacts.marketing_version} (build #{build_version}) for macOS.

          Download #{@artifacts.zip_name}, unzip it, and move #{File.basename(@artifacts.app)} to Applications.
        NOTES
      end
      sh! "gh", "release", "create", tag, *assets,
        "--repo", @settings.gh_repo, "--draft", "--verify-tag",
        "--title", "#{@artifacts.app_name} #{@artifacts.marketing_version}", "--notes-file", notes
    end
    sh! "gh", "release", "edit", tag, "--repo", @settings.gh_repo,
      "--draft=false", "--prerelease=false", "--latest"
    ok "GitHub release #{tag} published with #{@artifacts.zip_name} and appcast.xml"
    note "Sparkle feed: https://github.com/#{@settings.gh_repo}/releases/latest/download/appcast.xml"
  end
end

class ReleaseSource
  def initialize(settings, artifacts, github_release)
    @root = File.dirname(settings.path)
    @artifacts = artifacts
    @github_release = github_release
  end

  def commit
    capture!("git", "-C", @root, "rev-parse", "HEAD").strip
  end

  def clean!
    paths = ["."]
    relative_build = Pathname.new(@artifacts.build_dir).relative_path_from(Pathname.new(@root)).to_s
    if relative_build == "."
      halt("BUILD_DIR must not be the project root")
    elsif relative_build != ".." && !relative_build.start_with?("../")
      tracked_build_files = capture!("git", "-C", @root, "ls-files", "--", relative_build)
      halt("BUILD_DIR contains tracked source files; choose a separate build directory") unless tracked_build_files.empty?
      paths << ":(exclude)#{relative_build}"
    end
    changes = capture!("git", "-C", @root, "status", "--porcelain", "--untracked-files=all", "--", *paths)
    halt("source has uncommitted or untracked changes; commit or ignore them before releasing:\n#{changes}") unless changes.empty?
  end

  def record!
    File.write(@artifacts.source_commit_path, "#{commit}\n")
  end

  def verify!
    clean!
    halt("missing archive source record — run `rake release:run:archive` first") unless File.file?(@artifacts.source_commit_path)
    current_commit = commit
    halt("checkout changed since archive; rebuild the release") unless File.read(@artifacts.source_commit_path).strip == current_commit
    remote_commit = @github_release.remote_tag_commit(@artifacts.tag)
    halt("remote tag #{@artifacts.tag} does not point to archived commit #{current_commit}") unless remote_commit == current_commit
  end
end

RELEASE_SETTINGS = ReleaseSettings.new("release.json")
RELEASE_ARTIFACTS = ReleaseArtifacts.new(RELEASE_SETTINGS)
GITHUB_RELEASE = GitHubRelease.new(RELEASE_SETTINGS, RELEASE_ARTIFACTS)
RELEASE_SOURCE = ReleaseSource.new(RELEASE_SETTINGS, RELEASE_ARTIFACTS, GITHUB_RELEASE)

ROOT = __dir__
Dir.chdir(ROOT)

# ---- helpers ---------------------------------------------------------------

# ANSI SGR color codes, named so the tasks below read as ok/warn/halt/step
# instead of raw \e[..m sequences.
GREEN = 32 # success lines
YELLOW = 33 # caution lines
RED = 31 # failure messages
CYAN = 36 # step banners and echoed commands

# Wrap the string in an ANSI color and reset. One place for the escape codes.
class String
  def colorize(number)
    "\e[#{number}m#{self}\e[0m"
  end
end

def ok(text)
  puts "✓ #{text}".colorize(GREEN) # green success line
end

def warn(text)
  puts text.colorize(YELLOW) # yellow caution line
end

def note(text)
  puts "  #{text}" # plain, indented secondary hint
end

def halt(text)
  abort text.colorize(RED) # red message, then abort the run
end

def setup_confirm(question)
  loop do
    print "#{question} [Y/n]: "
    $stdout.flush
    answer = $stdin.gets
    halt("setup cancelled") unless answer
    response = answer.strip.downcase
    return true if ["", "y", "yes"].include?(response)
    return false if ["n", "no"].include?(response)
    warn "  enter yes or no"
  end
end

# Print a cyan banner announcing the step about to run.
def step(text)
  puts "\n▶ #{text}".colorize(CYAN)
end

# Run a command, echoing it first. Raises (aborting the rake run) on failure.
def sh!(*args)
  puts "$ #{args.map { |a| Shellwords.escape(a) }.join(" ")}".colorize(CYAN)
  system(*args) || halt("command failed: #{args.first}")
end

# Capture stdout of a command, aborting on failure.
def capture!(*args)
  out = IO.popen(args, &:read)
  halt("command failed: #{args.first}") unless $?.success?
  out
end

def notary_profile_exists?(profile = RELEASE_SETTINGS.notary_profile)
  # `notarytool history` succeeds only if the named keychain profile resolves.
  system("xcrun", "notarytool", "history", "--keychain-profile", profile,
    out: File::NULL, err: File::NULL)
end

def upstream_rakefile
  uri = URI(UPSTREAM_RAKEFILE)
  response = Net::HTTP.start(uri.host, uri.port, use_ssl: true, open_timeout: 10, read_timeout: 30) do |http|
    http.get(uri.request_uri)
  end
  halt("Could not download #{UPSTREAM_RAKEFILE}: HTTP #{response.code}") unless response.is_a?(Net::HTTPSuccess)
  response.body
rescue => error
  halt("Could not download #{UPSTREAM_RAKEFILE}: #{error.message}")
end

# ---- tasks -----------------------------------------------------------------

desc "Full release: preflight → archive → export → zip → notarize → appcast → GitHub publish"
task release: %w[
  release:run:preflight
  release:run:archive
  release:run:export
  release:run:zip
  release:run:notarize
  release:run:appcast
  release:run:github
] do
  ok "release #{RELEASE_ARTIFACTS.tag} published"
end

namespace :release do
  desc "Interactively configure release.json and notarization"
  task :setup do
    path = RELEASE_SETTINGS.path
    config = RELEASE_SETTINGS.config.dup
    manifest_path = RELEASE_SETTINGS.manifest
    manifest = File.file?(manifest_path) ? File.read(manifest_path) : ""
    projects = Dir.glob(File.join(ROOT, "*.xcworkspace")) + Dir.glob(File.join(ROOT, "*.xcodeproj"))
    project_basenames = projects.map { |project| File.basename(project).sub(/\.(xcworkspace|xcodeproj)\z/, "") }.uniq
    inferred_scheme = project_basenames.first if project_basenames.one?
    project_names = manifest.scan(/\bname:\s*"([^"]+)"/).flatten.uniq
    inferred_scheme ||= project_names.first if project_names.one?
    bundle_ids = manifest.scan(/bundleId:\s*"([^"]+)"/).flatten +
      manifest.scan(/"PRODUCT_BUNDLE_IDENTIFIER"\s*:\s*"([^"]+)"/).flatten
    bundle_ids.uniq!
    inferred_account = bundle_ids.first if bundle_ids.one? && !bundle_ids.first.match?(/[\\$]/)

    defaults = {
      "SCHEME" => inferred_scheme,
      "SPARKLE_ACCOUNT" => inferred_account
    }
    labels = {
      "SCHEME" => "Xcode scheme to archive",
      "TEAM_ID" => "Apple Developer team ID",
      "NOTARY_PROFILE" => "notarytool Keychain profile name",
      "SPARKLE_ACCOUNT" => "Sparkle signing key account"
    }

    labels.each do |key, label|
      next if config.key?(key)

      scheme = config["SCHEME"] || ENV["SCHEME"] || inferred_scheme
      defaults["NOTARY_PROFILE"] = "#{scheme.downcase.gsub(/[^a-z0-9]+/, "-")}-notary" if scheme.is_a?(String) && !scheme.empty?
      default = ENV[key].to_s.strip.empty? ? defaults[key] : ENV[key]
      loop do
        print "#{key} (#{label})#{default ? " [#{default}]" : ""}: "
        $stdout.flush
        answer = $stdin.gets
        halt("setup cancelled; release.json was not changed") unless answer
        value = answer.strip
        value = default if value.empty?
        if value && !value.empty?
          config[key] = value
          break
        end
        warn "  #{key} is required"
      end
    end

    if config == RELEASE_SETTINGS.config
      ok "release.json already has all required keys"
    else
      Tempfile.create([".release-", ".json"], ROOT) do |file|
        file.write("#{JSON.pretty_generate(config)}\n")
        file.flush
        File.chmod(File.stat(path).mode & 0o7777, file.path) if File.exist?(path)
        File.rename(file.path, path)
      end
      ok "saved #{path}"
    end

    gitignore = File.join(ROOT, ".gitignore")
    ignored = File.file?(gitignore) && File.foreach(gitignore).any? { |line| line.strip == "/release.json" }
    unless ignored
      if setup_confirm("Add /release.json to .gitignore?")
        content = File.file?(gitignore) ? File.binread(gitignore) : ""
        File.open(gitignore, "a") do |file|
          file.write("\n") unless content.empty? || content.end_with?("\n")
          file.write("/release.json\n")
        end
        ok "added /release.json to .gitignore"
      else
        note "Add /release.json to .gitignore before releasing."
      end
    end

    profile = ENV.fetch("NOTARY_PROFILE", config["NOTARY_PROFILE"])
    if notary_profile_exists?(profile)
      ok "notarytool profile \"#{profile}\" already exists"
      next
    end

    if setup_confirm("Create notarytool Keychain profile \"#{profile}\" now?")
      apple_id = ENV["APPLE_ID"] || config["APPLE_ID"]
      if apple_id.to_s.strip.empty?
        loop do
          print "APPLE_ID (Apple ID email): "
          $stdout.flush
          answer = $stdin.gets
          halt("setup cancelled; release.json was saved") unless answer
          apple_id = answer.strip
          break unless apple_id.empty?
          warn "  APPLE_ID is required"
        end
      end
      puts "Generate an app-specific password at https://appleid.apple.com → Sign-In and Security."
      # notarytool prompts for the password itself; it never enters release.json.
      sh! "xcrun", "notarytool", "store-credentials", profile,
        "--apple-id", apple_id,
        "--team-id", ENV.fetch("TEAM_ID", config["TEAM_ID"])
      ok "profile \"#{profile}\" saved; `rake release:run:notarize` can now notarize"
    else
      note "Run `rake release:setup` later to create the notarytool profile."
    end
  end

  namespace :meta do
    desc "show the diff between this Rakefile and the upstream main branch"
    task :diff do
      output, errors, status = Open3.capture3("diff", "-u", "--label", "local/Rakefile",
        "--label", "upstream/Rakefile", File.join(ROOT, "Rakefile"), "-",
        stdin_data: upstream_rakefile)
      halt("diff failed: #{errors.strip}") unless [0, 1].include?(status.exitstatus)
      print output
      ok "Rakefile matches upstream" if status.success?
    end

    desc "replace this Rakefile with the upstream main branch version"
    task :upgrade do
      path = File.join(ROOT, "Rakefile")
      content = upstream_rakefile
      if File.binread(path) == content
        ok "Rakefile already matches upstream"
        next
      end

      Tempfile.create([".Rakefile-", ".tmp"], ROOT) do |upstream|
        upstream.write(content)
        upstream.flush
        halt("Upstream Rakefile has invalid Ruby syntax") unless system(RbConfig.ruby, "-c", upstream.path, out: File::NULL)
        File.chmod(File.stat(path).mode & 0o7777, upstream.path)
        File.rename(upstream.path, path)
      end
      ok "Rakefile replaced with #{UPSTREAM_RAKEFILE}"
    end
  end

  desc "Bump the marketing version (VERSION=x.y) and increment the build number"
  task :bump do
    version = ENV["VERSION"]
    halt("set VERSION, e.g. `rake release:bump VERSION=1.1`") if version.to_s.strip.empty?
    halt("VERSION must look like 1.1 or 1.2.3") unless version.match?(/\A\d+(\.\d+){1,2}\z/)

    # Project.swift is authoritative; Tuist regenerates the Xcode project.
    manifest = RELEASE_SETTINGS.manifest
    halt("missing Tuist manifest: #{manifest}") unless File.file?(manifest)
    text = File.read(manifest)
    builds = text.scan(/"CURRENT_PROJECT_VERSION"\s*:\s*"(\d+)"/).flatten.map(&:to_i)
    halt("no CURRENT_PROJECT_VERSION found in #{manifest}") if builds.empty?
    halt("no MARKETING_VERSION found in #{manifest}") unless text.match?(/"MARKETING_VERSION"\s*:\s*"[^"]+"/)
    next_build = [builds.max, GITHUB_RELEASE.highest_published_build(GITHUB_RELEASE.releases)].max + 1
    text = text.gsub(/"CURRENT_PROJECT_VERSION"\s*:\s*"\d+"/, %("CURRENT_PROJECT_VERSION": "#{next_build}"))
    text = text.gsub(/"MARKETING_VERSION"\s*:\s*"[^"]+"/, %("MARKETING_VERSION": "#{version}"))
    File.write(manifest, text)
    ok "marketing version #{version}, build #{next_build}"
    note "Review and commit #{manifest} before releasing."
  end

  namespace :run do
    desc "check that the tools, keys, and repo state a release needs are in place"
    task :preflight do
      step "preflight — checking the release environment"

      # Collect every problem, then report them together, so one `rake release`
      # surfaces all the fixes at once instead of failing on the first missing key.
      problems = []
      ask = ->(label, ok_cond) { ok_cond ? ok(label) : (problems << label) }

      RELEASE_SETTINGS.scheme
      RELEASE_SETTINGS.team_id
      RELEASE_SETTINGS.notary_profile
      RELEASE_SETTINGS.sparkle_account
      RELEASE_SETTINGS.gh_repo
      RELEASE_SOURCE.clean!
      notes = RELEASE_SETTINGS.release_notes
      halt("missing RELEASE_NOTES file: #{notes}") if notes && !File.file?(notes)
      sh! "tuist", "install"

      # xcodebuild for archive/export.
      ask.call("xcodebuild present", system("xcodebuild", "-version", out: File::NULL, err: File::NULL))

      # Sparkle tools (generate_appcast for the appcast, generate_keys to read the
      # EdDSA key) are installed by Tuist.
      bin = RELEASE_SETTINGS.sparkle_bin_dir
      ask.call("Sparkle tools resolved (run tuist install or set SPARKLE_BIN_DIR)", bin && File.exist?(File.join(bin, "generate_appcast")))

      # The appcast is signed with the EdDSA private key in the Keychain.
      # generate_keys -p prints the public key and exits 0 only if the key exists.
      keys = bin && File.join(bin, "generate_keys")
      ask.call("Sparkle EdDSA signing key in Keychain (run `#{keys || "generate_keys"} --account #{RELEASE_SETTINGS.sparkle_account}` once)", keys && File.exist?(keys) &&
             system(keys, "--account", RELEASE_SETTINGS.sparkle_account, "-p", out: File::NULL, err: File::NULL))

      # notarytool keychain profile for the notarize step.
      ask.call("notarytool profile \"#{RELEASE_SETTINGS.notary_profile}\" saved (run `rake release:setup`)", notary_profile_exists?)

      # gh authenticated for creating the GitHub release.
      ask.call("gh authenticated (run `gh auth login`)", system("gh", "auth", "status", out: File::NULL, err: File::NULL))

      # Managed Developer ID signing usually can't be listed on the CLI, so a
      # missing cert here is a heads-up, not a failure — the export may still work.
      unless system("sh", "-c",
        "security find-identity -v -p codesigning | grep -q 'Developer ID Application'",
        out: File::NULL, err: File::NULL)
        warn "  ⚠ no 'Developer ID Application' cert listed on the CLI — fine if Xcode manages signing, but export will fail if it truly can't sign"
      end

      halt("preflight found problems:\n  - #{problems.join("\n  - ")}") unless problems.empty?
      ok "preflight passed — the release environment looks ready"
    end

    desc "archive the app (xcodebuild archive)"
    task :archive do
      step "archiving the app"
      scheme = RELEASE_SETTINGS.scheme
      RELEASE_SOURCE.clean!
      FileUtils.mkdir_p(RELEASE_ARTIFACTS.build_dir)
      # Invalidate downstream artifacts before building, including on failure.
      # Otherwise a resumed publish could pair an old ZIP with this source record.
      FileUtils.rm_rf([RELEASE_ARTIFACTS.archive, RELEASE_ARTIFACTS.export_dir, RELEASE_ARTIFACTS.dist_dir])
      FileUtils.rm_f([RELEASE_ARTIFACTS.source_commit_path, File.join(RELEASE_ARTIFACTS.build_dir, "notarization.json")])
      sh! "tuist", "install"
      sh! "tuist", "generate", "--no-open"
      sh! "tuist", "xcodebuild", "archive",
        "-workspace", RELEASE_SETTINGS.workspace,
        "-scheme", scheme,
        "-configuration", RELEASE_SETTINGS.configuration,
        "-destination", "generic/platform=macOS",
        "-archivePath", RELEASE_ARTIFACTS.archive,
        "-derivedDataPath", File.join(RELEASE_ARTIFACTS.build_dir, "DerivedData"),
        "-allowProvisioningUpdates",
        "ARCHS=arm64 x86_64", "ONLY_ACTIVE_ARCH=NO"
      RELEASE_SOURCE.record!
      ok "archived → #{RELEASE_ARTIFACTS.archive}"
    end

    desc "export a Developer ID-signed .app from the archive"
    task :export do
      step "exporting a Developer ID-signed .app"
      halt("missing archive — run `rake release:run:archive` first") unless File.exist?(RELEASE_ARTIFACTS.archive)
      team_id = RELEASE_SETTINGS.team_id
      # A new export must be zipped, notarized, and signed again before publishing.
      FileUtils.rm_rf([RELEASE_ARTIFACTS.export_dir, RELEASE_ARTIFACTS.dist_dir])
      FileUtils.rm_f(File.join(RELEASE_ARTIFACTS.build_dir, "notarization.json"))

      # method=developer-id reuses Xcode's managed Developer ID signing, so this
      # works even when `security find-identity` can't list the cert on the CLI.
      options = File.join(RELEASE_ARTIFACTS.build_dir, "ExportOptions.plist")
      File.write(options, <<~PLIST)
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
        <plist version="1.0">
        <dict>
            <key>method</key><string>developer-id</string>
            <key>signingStyle</key><string>automatic</string>
            <key>teamID</key><string>#{REXML::Text.normalize(team_id)}</string>
            <key>signingCertificate</key><string>Developer ID Application</string>
        </dict>
        </plist>
      PLIST

      # Tuist does not support the export action.
      sh! "xcodebuild", "-exportArchive",
        "-archivePath", RELEASE_ARTIFACTS.archive,
        "-exportPath", RELEASE_ARTIFACTS.export_dir,
        "-allowProvisioningUpdates",
        "-exportOptionsPlist", options
      sh! "codesign", "--verify", "--deep", "--strict", "--verbose=2", RELEASE_ARTIFACTS.app
      ok "exported → #{RELEASE_ARTIFACTS.app} (#{RELEASE_ARTIFACTS.marketing_version}, build #{RELEASE_ARTIFACTS.build_version})"
    end

    desc "zip the exported .app for distribution"
    task :zip do
      step "zipping the .app"
      FileUtils.mkdir_p(RELEASE_ARTIFACTS.dist_dir)
      FileUtils.rm_f(RELEASE_ARTIFACTS.zip_path)
      # ditto preserves the bundle's symlinks/metadata; Sparkle expects a clean zip.
      sh! "ditto", "-c", "-k", "--sequesterRsrc", "--keepParent", RELEASE_ARTIFACTS.app, RELEASE_ARTIFACTS.zip_path
      ok "zipped → #{RELEASE_ARTIFACTS.zip_path}"
    end

    desc "notarize the zip with notarytool and staple the .app"
    task :notarize do
      step "notarizing and stapling"
      halt("missing #{RELEASE_ARTIFACTS.zip_path} — run `rake release:run:zip` first") unless File.exist?(RELEASE_ARTIFACTS.zip_path)

      halt("No notarytool profile #{RELEASE_SETTINGS.notary_profile.inspect}; run `rake release:setup`.") unless notary_profile_exists?
      result = capture!("xcrun", "notarytool", "submit", RELEASE_ARTIFACTS.zip_path,
        "--keychain-profile", RELEASE_SETTINGS.notary_profile, "--wait", "--output-format", "json")
      File.write(File.join(RELEASE_ARTIFACTS.build_dir, "notarization.json"), result)
      submission = JSON.parse(result)
      halt("Notarization #{submission["status"]}; use `xcrun notarytool log #{submission["id"]} --keychain-profile #{RELEASE_SETTINGS.notary_profile}`.") unless submission["status"] == "Accepted"
      # Staple the ticket onto the .app, then re-zip so the distributed zip carries it.
      sh! "xcrun", "stapler", "staple", RELEASE_ARTIFACTS.app
      sh! "xcrun", "stapler", "validate", RELEASE_ARTIFACTS.app
      FileUtils.rm_f(RELEASE_ARTIFACTS.zip_path)
      sh! "ditto", "-c", "-k", "--sequesterRsrc", "--keepParent", RELEASE_ARTIFACTS.app, RELEASE_ARTIFACTS.zip_path
      sh! "codesign", "--verify", "--deep", "--strict", "--verbose=2", RELEASE_ARTIFACTS.app
      sh! "spctl", "--assess", "--type", "execute", "--verbose=2", RELEASE_ARTIFACTS.app
      File.write(RELEASE_ARTIFACTS.checksum_path, "#{Digest::SHA256.file(RELEASE_ARTIFACTS.zip_path).hexdigest}  #{RELEASE_ARTIFACTS.zip_name}\n")
      ok "notarized + stapled; re-zipped → #{RELEASE_ARTIFACTS.zip_path}"
    end

    desc "generate/update the EdDSA-signed appcast.xml for this version"
    task :appcast do
      step "generating the signed appcast.xml"
      halt("missing #{RELEASE_ARTIFACTS.zip_path} — run earlier steps first") unless File.exist?(RELEASE_ARTIFACTS.zip_path)

      sh! "xcrun", "stapler", "validate", RELEASE_ARTIFACTS.app
      public_key = capture!(File.join(RELEASE_SETTINGS.sparkle_bin_dir, "generate_keys"), "--account", RELEASE_SETTINGS.sparkle_account, "-p").strip
      halt("Sparkle Keychain key does not match the app's public key") unless public_key == RELEASE_ARTIFACTS.plist("SUPublicEDKey")
      expected_feed = "https://github.com/#{RELEASE_SETTINGS.gh_repo}/releases/latest/download/appcast.xml"
      halt("App feed URL does not match GH_REPO") unless RELEASE_ARTIFACTS.plist("SUFeedURL") == expected_feed

      sh! RELEASE_SETTINGS.generate_appcast_bin,
        "--account", RELEASE_SETTINGS.sparkle_account,
        "--versions", RELEASE_ARTIFACTS.build_version,
        "--maximum-deltas", "0",
        "--download-url-prefix", RELEASE_ARTIFACTS.download_prefix,
        "-o", RELEASE_ARTIFACTS.appcast,
        RELEASE_ARTIFACTS.dist_dir
      ok "appcast.xml updated for #{RELEASE_ARTIFACTS.marketing_version} (build #{RELEASE_ARTIFACTS.build_version})"
      note "enclosure URL prefix: #{RELEASE_ARTIFACTS.download_prefix}"
    end

    desc "publish a regular GitHub release with the ZIP, checksum, and appcast"
    task :github do
      step "publishing the GitHub release"
      GITHUB_RELEASE.publish!(RELEASE_SOURCE)
    end
  end
end
