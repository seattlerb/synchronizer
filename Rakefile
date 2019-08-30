# -*- ruby -*-

require "open-uri"
require "yaml"
require "rubygems"
require "json"

ENV["P4CONFIG"] = ".p4config"
ENV["P4CHARSET"] = "utf8" # let's hope this works... :/

TRACE = !!Rake.application.options.trace
RakeFileUtils.verbose_flag = TRACE
DEVNULL = TRACE ? "" : "> /dev/null"
FAST = ENV['FAST']

COLLABORATORS = %w(zenspider)
GITHUB_USER = `git config github.user`.chomp
GITHUB_PASS = File.read File.expand_path "~/.gitconfig.passwd"
GITHUB_ORG    = "seattlerb"
GITHUB_TOKEN  = IO.read("token.txt").strip rescue nil
GIT_P4        = File.expand_path "vendor/git-p4"

ALL_PROJECTS = IO.read("projects.txt").
  split("\n").
  reject { |p| p =~ /^#/ }.
  sort_by { |p| p.downcase }

def git_p4 *args
  sh "python #{GIT_P4} #{args.join ' '} #{DEVNULL}"
end

def github url_suffix, options = {}
  url  = "https://api.github.com/#{url_suffix}"

  flags = [  "-H \"Authorization: token #{GITHUB_TOKEN}\"",
             "-isS",
             '-H "Content-Type: application/json"',
             '-H "Accept: application/json"',
             '-X POST'].join " "

  data = JSON.dump options

  cmd = "curl #{flags} -d '#{data}' #{url} #{DEVNULL}"
  sh cmd
end

def link_hash link
  Hash[link.scan(/<([^>]+)>; rel="([^"]+)"/).map(&:reverse)]
end

def paged_json_array url
  result = []

  begin
    warn "url: #{url}" if TRACE

    req  = URI.parse(url).open "User-Agent" => "omfg.seattlerb.org"
    link = link_hash req.meta["link"]
    url  = link["next"]

    result.concat JSON.load(req.read)
  end while url

  result
end

def git *args
  sh "git #{args.join ' '} #{DEVNULL}"
end

def projects
  return ALL_PROJECTS unless ENV["PROJECTS"]
  ENV["PROJECTS"].split ","
end

desc "Pull projects from Perforce."
task :default => :pull

desc "Pull projects from Perforce and push to GitHub."
task :sync => %w(pull push)

def git_dirty
  if `git diff`.lines.size != 0 and not ENV["DIRTY"] then
    warn "git is dirty. skipping"
    true
  end
end

task :pull do
  next if git_dirty

  projects.each do |name|
    warn "* Pulling #{name} from Perforce." if TRACE

    src = "//src/#{name}/dev@all"

    name.downcase!
    dest = "projects/#{name}"

    unless File.directory? dest
      mkdir_p dest

      Dir.chdir dest do
        git_p4 :clone, src, "."
        git    "remote add origin git@github.com:#{GITHUB_ORG}/#{name}.git"
      end
    else
      Dir.chdir dest do
        git_p4 :sync
        git    "rebase p4/master"
      end
    end
  end
end

task :push do
  next if git_dirty

  warn "Getting repos..." if TRACE

  url   = "https://api.github.com/orgs/#{GITHUB_ORG}/repos"
  repos = paged_json_array(url).map { |r| r["name"] }
  p repos.sort if TRACE

  projects.each do |name, project|
    name.downcase!

    unless repos.include? name
      warn "  - Creating a new repo!" if TRACE

      github "orgs/#{GITHUB_ORG}/repos", :name => name
    end

    Dir.chdir("projects/#{name}") do
      should_push = !repos.include?(name) ||
       !system("git diff --quiet origin/master")
      should_push_tags = changelog_to_tags

      if should_push then
        warn "Pushing #{name} to GitHub." if TRACE
        sleep rand(30) unless FAST
        git "push origin master"
      end

      if should_push_tags then
        warn "Pushing tags #{name} to GitHub." if TRACE
        git "push --tags origin master"
      end
    end
  end
end

desc "wtf"
task :wtf do
  base = "~/Work/p4/zss/src/seattlerb_dashboard/dev/lib"
  $: << File.expand_path(base)
  require "seattlerb_projects"
  File.open("projects.txt", "w") do |f|
    f.puts SeattlerbProjects.new.projects.flatten.sort.join("\n")
  end
end

def changelog_to_tags
  cmd = %w[git log
           --abbrev-commit
           --pretty=tformat:"~~~%n%h: %s"
           -p
           --
           History.rdoc
           History.md
           History.txt
          ].join " "

  changes = `#{cmd}`.split(/~~~\n/).drop 1
  seen    = `git tag`.lines.map(&:chomp).map { |k| [k, true] }.to_h

  versions = changes.map { |change|
    lines = change.lines
    sha, ver = lines[0], lines.grep(/^\+=+ \d/).first

    next unless sha =~ /prep/i
    next unless ver

    sha = sha[/^\w+/]
    ver, date = ver.split.values_at(1, 3)

    next unless date =~ /^\d\d\d\d-\d\d-\d\d$/
    next unless ver  =~ /^\d+\.\d+\.\d+/

    [date, "v#{ver}", sha]
  }.compact

  additions = versions.sort.map { |date, ver, sha|
    next if seen[ver]
    cmd = "git tag %-12s %s # %s" % [ver, sha, date]
    warn cmd
    system cmd
  }.compact

  ! additions.empty?
end
