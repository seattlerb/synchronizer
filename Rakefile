# -*- ruby -*-

require "open-uri"
require "yaml"
require "rubygems"
require "json"

ENV["P4CHARSET"] = "utf8" # let's hope this works... :/

TRACE = !!Rake.application.options.trace
RakeFileUtils.verbose_flag = TRACE
DEVNULL = TRACE ? "" : "> /dev/null"
FAST = ENV['FAST']

COLLABORATORS = %w(zenspider)
GITHUB_USER   = "zenspider"
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
  scheme = options.delete(:scheme) || "https"
  fields = options.collect { |k,v| "-F '#{k}=#{v}'" }.join ' '
  url    = "#{scheme}://api.github.com/#{url_suffix}"
  auth   = "#{GITHUB_USER}/token:#{GITHUB_TOKEN}"

  sh "curl -u #{auth} -isS #{fields} #{url} #{DEVNULL}"
end

def link_hash link
  Hash[link.scan(/<([^>]+)>; rel="([^"]+)"/).map(&:reverse)]
end

def paged_json_array url
  result = []

  begin
    warn url
    req  = URI.parse(url).open
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

task :pull do
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
  url   = "https://api.github.com/orgs/#{GITHUB_ORG}/repos"
  repos = paged_json_array(url).map { |r| r["name"] }
  p repos.sort if TRACE

  projects.each do |name, project|
    name.downcase!

    warn "Pushing #{name} to GitHub." if TRACE

    unless repos.include? name
      warn "  - Creating a new repo!" if TRACE

      github "/orgs/#{GITHUB_ORG}/repos", :name => name
    end

    Dir.chdir("projects/#{name}") do
      should_push = !repos.include?(name) ||
       !system("git diff --quiet origin/master")

      if should_push then
        sleep rand(30) unless FAST
        git "push origin master"
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
