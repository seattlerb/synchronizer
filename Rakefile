COLLABORATORS = %w(jbarnette zenspider)
GITHUB_LOGIN  = "seattlerb"
GITHUB_TOKEN  = IO.read("token.txt").strip
GIT_P4        = File.expand_path "vendor/git-p4"

ALL_PROJECTS = IO.read("projects.txt").
  split("\n").
  reject { |p| p =~ /^#/ }.
  sort_by { |p| p.downcase }

def git_p4 *args
  sh "python #{GIT_P4} #{args.join(' ')}"
end

def github url_suffix, options
  scheme = options.delete(:scheme) || "https"

  options.merge!({ :login => GITHUB_LOGIN, :token => GITHUB_TOKEN })
  fields = options.collect { |k,v| "-F '#{k}=#{v}'" }

  sh "curl #{fields.join ' '} #{scheme}://github.com/#{url_suffix} > /dev/null"
end

def git_in dir, *args
  Dir.chdir(dir) { sh ["git", *args].join(" ") }
end

def github_project_exists? name
  url = "http://github.com/seattlerb/#{name}/tree/master"
  `curl -s -I #{url}` =~ /200 OK/
end

def projects
  return ALL_PROJECTS unless ENV["PROJECTS"]
  ENV["PROJECTS"].split ","
end

task :default do
  p projects
end

desc "Pull changes from Perforce and push to GitHub."
task :sync => %w(pull push)

task :pull do
  projects.each do |name|
    src = "//src/#{name}/dev@all"

    name.downcase!
    dest = "projects/#{name}"

    unless File.directory? dest
      mkdir_p dest
      git_p4 :clone, src, dest
      git_in dest, "add origin git@github.com:seattlerb/#{name}.git"
    else
      git_in dest, "rebase p4/master"
    end
  end
end

task :push do
  projects.each do |name, project|
    name.downcase!

    unless github_project_exists? name
      github :repositories, :scheme => :http,
        "repository[name]" => name

      COLLABORATORS.each do |collaborator|
        github "seattlerb/#{name}/edit/add_member", :member => collaborator
      end
    end

    git_in "projects/#{name}", "push origin master"
  end
end
