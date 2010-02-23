require "open-uri"
require "yaml"

TRACE = Rake.application.options.trace
RakeFileUtils.verbose_flag = TRACE
DEVNULL = TRACE ? "" : "> /dev/null"

COLLABORATORS = %w(jbarnette zenspider)
GITHUB_LOGIN  = "seattlerb"
GITHUB_TOKEN  = IO.read("token.txt").strip
GIT_P4        = File.expand_path "vendor/git-p4"

ALL_PROJECTS = IO.read("projects.txt").
  split("\n").
  reject { |p| p =~ /^#/ }.
  sort_by { |p| p.downcase }

def git_p4 *args
  sh "python #{GIT_P4} #{args.join ' '} #{DEVNULL}"
end

def github url_suffix, options
  options.merge! :login => GITHUB_LOGIN, :token => GITHUB_TOKEN

  scheme = options.delete(:scheme) || "https"
  fields = options.collect { |k,v| "-F '#{k}=#{v}'" }.join ' '
  url    = "#{scheme}://github.com/#{url_suffix}"

  sh "curl -s #{fields} #{url} #{DEVNULL}"
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
        git    "remote add origin git@github.com:seattlerb/#{name}.git"
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
  url   = "http://github.com/api/v2/yaml/repos/show/seattlerb"
  repos = YAML.load(open(url).read)["repositories"].map { |r| r[:name] }

  projects.each do |name, project|
    name.downcase!

    warn "Pushing #{name} to GitHub." if TRACE

    unless repos.include? name
      warn "  - Creating a new repo!" if TRACE
      
      github :repositories, :scheme => :http,
        "repository[name]" => name

      COLLABORATORS.each do |collaborator|
        github "seattlerb/#{name}/edit/add_member", :member => collaborator
      end
    end

    Dir.chdir("projects/#{name}") do
      should_push = !repos.include?(name) ||
        !`git diff origin/master`.strip.empty?

      git "push origin master" if should_push
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
