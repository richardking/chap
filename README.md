# Chap

chef + capistrano = chap: deploy your app with either chef or capistrano.  This was written to solve the issue between having 2 deployment systems that are very similar but not exactly the same.  With chap you can deploy to a single server by running one command: 

<pre>
$ chap deploy
</pre>

The same command is called whether you're using chef or capistrano for deployemnt.  The chap deploy command does the heavily lifting and manages the deploy instead of capistrano or chef.

## Requirements

<pre>
$ gem install chap
</pre>

## Usage

Chap command is meant to be executed on the server which you want to deploy the code.  

## Setup

Chap requires 3 configuration files: chap.yml, chap.json and node.json.

* chap.json: contains capistrano-like configuration settings.
* node.json: is intended to be the same file that chef solo uses and contains instance specific information, like node[:instance_role].
: chap.yml: The paths of the chap.json and node.json are configurable via the chap.yml file.

Here are examples of the starter setup files that you can generate via: 

<pre>
$ chap setup -o /etc/chef
</pre>

<pre>
$ cat /etc/chef/chap.yml
chap: /etc/chef/chap.json
node: /etc/chef/node.json
$ cat /etc/chef/chap.json
{
  "repo": "git@github.com:tongueroo/chapdemo.git",
  "branch": "master",
  "application": "chapdemo",
  "deploy_to": "/data/chapdemo",
  "strategy": "git",
  "keep": 5,
  "user": "deploy",
  "group": "deploy"
}
$ cat /etc/chef/node.json
{
  "environment": "staging",
  "application": "chapdemo",
  "instance_role": "app"
}
</pre>

### Deploy sequence

The deploy sequence is based on the sequence from the capistrano and chef deploy resource provider.

1. Download code to /data/[app]/releases/[timestamp]
2. Run chap/deploy hook
3. Symlink /data/[app]/releases/[timestamp] to /data/[app]/current
4. Run chap/restart hook
5. Clean up old releases

On the server:

<pre>
$ chap deploy
</pre>

From capistrano, on local or deploy box:

<pre>
$ cap deploy # cap recipe calls "chap deploy"
</pre>

Chef Chap LWRP:

<pre>
chap_deploy # chef resource calls "chap deploy"
</pre>


Chap loads up information from node.json because it needs that information for hooks, which tend to work differently for different server roles.  For example, the chap/restart hook below will run "touch tmp/restart.txt" for an app role and will run "rvmsudo bluepill restart resque" for a resque role.  Example:

<pre>
$ cat chap/restart
#!/usr/bin/env ruby
restart = case node[:instance_role]
          when 'app'
            "touch tmp/restart.txt"
          when 'resque'
            "rvmsudo bluepill restart resque"
          end
run "cd #{current_path} && #{restart}"
</pre>

## Deploy Hooks

Define your deploy hooks in the chap folder of the project.  There are 2 deploy hooks.

* chap/deploy - runs after the code has been deployed by not yet symlinked
* chap/restart - runs after the code has been symlinked and app needs to be restarted

Deploy hooks get evaluated within the context of a chap deploy run and have some special variables and methods:

Special variables:

* node - contains data from /etc/chef/node.json.  Avaiable as mash.
* chap - contains data from /etc/chef/chap.json and some special variables added by chap.  Avaiable as mash.  Special variables: release_path, current_path, shared_path, cached_path.  The special variables are also available directly as methods.

Special methods:

* run - output the command to be ran and runs command.
* log - log messages to [shared_path]/chap/chap.log.