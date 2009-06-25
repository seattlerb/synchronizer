require "open-uri"
require "yaml"

COLLABORATORS = %w(jbarnette zenspider)
GITHUB_LOGIN  = "seattlerb"
GITHUB_TOKEN  = IO.read("token.txt").strip
GIT_P4        = File.expand_path "vendor/git-p4"

ALL_PROJECTS = IO.read("projects.txt").
  split("\n").
  reject { |p| p =~ /^#/ }.
  sort_by { |p| p.downcase }

def git_p4 *args
  sh "python #{GIT_P4} #{args.join ' '}"
end

def github url_suffix, options
  scheme = options.delete(:scheme) || "https"

  options.merge!({ :login => GITHUB_LOGIN, :token => GITHUB_TOKEN })
  fields = options.collect { |k,v| "-F '#{k}=#{v}'" }

  sh "curl #{fields.join ' '} #{scheme}://github.com/#{url_suffix} > /dev/null"
end

def git *args
  sh "git #{args.join ' '}"
end

def projects
  return ALL_PROJECTS unless ENV["PROJECTS"]
  ENV["PROJECTS"].split ","
end

task :default do
  y projects
end

desc "Pull changes from Perforce and push to GitHub."
task :sync => %w(pull push)

task :pull do
  projects.each do |name|
    warn "* Pulling #{name} from Perforce." if $DEBUG

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

    warn "Pushing #{name} to GitHub." if $DEBUG

    unless repos.include? name
      warn "  - Creating a new repo!" if $DEBUG

      github :repositories, :scheme => :http,
        "repository[name]" => name

      COLLABORATORS.each do |collaborator|
        github "seattlerb/#{name}/edit/add_member", :member => collaborator
      end
    end

    Dir.chdir("projects/#{name}") do
      unless `git diff origin/master`.strip.empty?
        git "push origin master"
      end
    end
  end
end
