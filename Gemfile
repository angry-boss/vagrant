source "https://rubygems.org"

gemspec

if File.exist?(File.expand_path("../../vagrant-spec", __FILE__))
  gem 'vagrant-spec', path: "../vagrant-spec"
else
  gem 'vagrant-spec', git: "https://github.com/angry-boss/vagrant-spec.git"
end
