# Prepare capistrano


## Modify Gemfile

	gem 'whenever', :require => false
	
	# Use unicorn as the app server
	group :production, :staging do
	  gem 'unicorn'
	end

	group :development do
	  gem 'capistrano', '~> 3.1.0'
	  gem 'capistrano-upload-config'
	  gem 'capistrano-rails', '~> 1.1'
	  gem 'capistrano3-unicorn'
	  gem 'capistrano3-nginx'
	  gem 'capistrano-bundler'
	  gem 'capistrano-rbenv', '~> 2.0'
	end

## Execute

	cap install
	wheneverize .

## Modify the following files:

### Capfile
	# Load DSL and Setup Up Stages
	require 'capistrano/setup'

	# Includes default deployment tasks
	require 'capistrano/deploy'

	require 'capistrano/bundler'
	require 'capistrano3/unicorn'
	require 'capistrano/nginx'


	require 'capistrano/rails'
	require 'capistrano/upload-config'

	# Loads custom tasks from `lib/capistrano/tasks' if you have any defined.
	Dir.glob('lib/capistrano/tasks/*.cap').each { |r| import r }

### lib/capistrano/tasks/nginx_management.cap

	namespace :nginx do
	  %w[start stop restart reload].each do |command|
	    desc "#{command} Nginx."
	    task command do
	      on roles(:web) do
	        sudo "service nginx #{command}"
	      end
	    end
	  end
	end

### config/deploy.rb

Change:

- imetricas_test
- dperezrada/imetricas_test

In the following file

	# config valid only for Capistrano 3.1
	lock '3.1.0'


	# Whenever config
	set :whenever_roles,        ->{ :db }
	set :whenever_command,      ->{ [:bundle, :exec, :whenever] }
	set :whenever_command_environment_variables, ->{ {} }
	set :whenever_identifier,   ->{ fetch :application }
	set :whenever_environment,  ->{ fetch :rails_env, "production" }
	set :whenever_variables,    ->{ "environment=#{fetch :whenever_environment}" }
	set :whenever_update_flags, ->{ "--update-crontab #{fetch :whenever_identifier} --set #{fetch :whenever_variables}" }
	set :whenever_clear_flags,  ->{ "--clear-crontab #{fetch :whenever_identifier}" }
	require "whenever/capistrano"

	set :application, 'imetricas_test'
	set :repo_url, 'git@github.com:dperezrada/imetricas_test.git'

	# Default branch is :master
	ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }

	set :deploy_to, '/var/www'

	# Default value for :scm is :git
	set :scm, :git
	set :branch, "master"

	# Default value for :pty is false
	set :pty, true

	# Default value for :linked_files is []
	set :linked_files, %w{config/database.yml }

	# Default value for linked_dirs is []
	set :linked_dirs, %w{bin log tmp/pids}

	set :keep_releases, 5

	set :rbenv_type, :user # or :system, depends on your rbenv setup
	set :rbenv_ruby, '2.1.2'
	set :rbenv_prefix, "RBENV_ROOT=#{fetch(:rbenv_path)} RBENV_VERSION=#{fetch(:rbenv_ruby)} #{fetch(:rbenv_path)}/bin/rbenv exec"
	set :rbenv_map_bins, %w{rake gem bundle ruby rails}
	set :rbenv_roles, :all # default value
	set :sidekiq_default_hooks, true

	# capistrano-upload-config configurations
	# Push actual configurations before the files are linked.
	before 'deploy:check:linked_files', 'config:push'
	set :config_files, fetch(:linked_files)
	set :config_example_prefix, '.default'

	set :default_env, { path: "~/.rbenv/shims:~/.rbenv/bin:$PATH" }

	namespace :deploy do

	  before :updating, :setup_rbenv do
	    require 'capistrano/rbenv'
	  end

	  desc 'Restart application'
	  task :restart do
	    on roles(:app), in: :sequence, wait: 5 do
	      # Your restart mechanism here, for example:
	      # execute :touch, release_path.join('tmp/restart.txt')
	    end
	  end

	   desc 'Runs rake db:seed for SeedMigrations data'
	  task :seed => [:set_rails_env] do
	    on primary fetch(:migration_role) do
	      within release_path do
	        with rails_env: fetch(:rails_env) do
	          execute :rake, "db:seed"
	        end
	      end
	    end
	  end

	  after 'deploy:migrate', 'deploy:seed'

	  after :publishing, :restart
	  after :publishing, "unicorn:restart"

	  after :restart, :clear_cache do
	    on roles(:web), in: :groups, limit: 3, wait: 10 do
	      # Here we can do anything such as:
	      # within release_path do
	      #   execute :rake, 'cache:clear'
	      # end
	    end
	  end
	end

### config/unicorn/production.rb
	worker_processes 4

	working_directory "/var/www/current" # available in 0.94.0+
	listen 8080, :tcp_nopush => true

	timeout 30

	pid "/var/www/current/tmp/pids/unicorn.pid"

	stderr_path "/var/www/shared/log/unicorn.stderr.log"
	stdout_path "/var/www/shared/log/unicorn.stdout.log"
	
## Config the files
 - config/deploy/production.rb
 - config/deploy/staging.rb

## Create file
 - config/database.production.yml
