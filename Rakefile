require "yaml"

PROJECTS = YAML.load(IO.read("projects.yml"))
GIT_P4   = File.expand_path "vendor/git-p4"

def git_p4 *args
  sh "python #{GIT_P4} #{args.join(' ')}"
end

task :default do
  p PROJECTS
end

task :sync => %w(pull push)

task :pull do
  PROJECTS.each do |name, project|
    dest = "projects/#{name}"

    unless File.directory? dest
      mkdir_p dest
      git_p4 :clone, "#{project['src']}@all", dest

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
    Dir.chdir("projects/#{name}") { sh "git push origin master" }
  end
end
