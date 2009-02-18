COLLABORATORS = %w(jbarnette zenspider)
GITHUB_LOGIN  = "seattlerb"
GITHUB_TOKEN  = IO.read("token.txt").strip
GIT_P4        = File.expand_path "vendor/git-p4"
PROJECTS      = IO.read("projects.txt").split("\n").reject { |p| p =~ /^#/ }

def git_p4 *args
  sh "python #{GIT_P4} #{args.join(' ')}"
end

def github url_suffix, options
  scheme = options.delete(:scheme) || "https"

  options.merge!({ :login => GITHUB_LOGIN, :token => GITHUB_TOKEN })
  fields = options.collect { |k,v| "-F '#{k}=#{v}'" }

  sh "curl #{fields.join ' '} #{scheme}://github.com/#{url_suffix} > /dev/null"
end

def github_project_exists? name
  `curl -I http://github.com/seattlerb/un/tree/master` =~ /200 OK/
end

task :default do
  p PROJECTS
end

desc "Pull changes from Perforce and push to GitHub."
task :sync => %w(pull push)

task :pull do
  PROJECTS.each do |name|
    dest = "projects/#{name}"

    unless File.directory? dest
      mkdir_p dest
      git_p4 :clone, "//src/#{name}/dev@all", dest

      Dir.chdir dest do
        sh "git remote add origin git@github.com:seattlerb/#{name}.git"
      end
    else
      Dir.chdir(dest) { git_p4 :sync }
    end
  end
end

task :push do
  PROJECTS.each do |name, project|
    unless github_project_exists? name
      github :repositories, :scheme => :http, "repository[name]" => name

      COLLABORATORS.each do |collaborator|
        github "seattlerb/#{name}/edit/add_member", :member => collaborator
      end
    end

    Dir.chdir("projects/#{name}") { sh "git push origin master" }
  end
end
