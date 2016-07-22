# Chef Intranet Scaffolding
What kind of scaffolding does it take to run Chef on a network with no Internet access?

### Busser / Serverspec
Historically, the trickiest part of operating Chef in an Air Gapped environment has been local development when using the `test-kitchen` verifier.
The default for a long time has been busser / serverspec (this changed to `inspec` in version 0.16.28 of `chef-dk`)

The trouble is that Busser is not Proxy friendly and does not allow use of an alternate GemSource to https://rubygems.org
 * https://github.com/test-kitchen/busser/issues/21
 * https://github.com/test-kitchen/busser/issues/33

The recommended approach is simply migrating over to `inspec` rather then try to tackle setting proxy environment variables, creating custom box images with the gems pre-installed, etc.

`inspec` does not reach out to the Internet for any additional gems to install. Additionally, migrating from `serverspec` is pretty straight forward. Keep in mind that `inspec` currently does not allow for nested `describe` blocks, other than that, it's resources are mostly the same with extra functionality and flexibility of custom extensions.

 * https://docs.chef.io/inspec_reference.html
 * https://github.com/chef/inspec

```js
# .kitchen.yml
verifier:
   name: inspec
   format: doc
```

The developers did an awesome job with the `help` sub-command to inspec - so there's no reason not to migrate!

Check it out.  I use it all the time myself :)

```sh
~/Devel/ChefProject/tmp$ inspec help
Commands:
  inspec archive PATH                # archive a profile to tar.gz (default) or zip
  inspec check PATH                  # verify all tests at the specified PATH
  inspec compliance SUBCOMMAND ...   # Chef Compliance commands
  inspec detect                      # detect the target OS
  inspec exec PATHS                  # run all test files at the specified PATH.
  inspec help [COMMAND]              # Describe available commands or one specific command
  inspec init TEMPLATE ...           # Scaffolds a new project
  inspec json PATH                   # read all tests in PATH and generate a JSON summary
  inspec shell                       # open an interactive debugging shell
  inspec supermarket SUBCOMMAND ...  # Supermarket commands
  inspec version                     # prints the version of this tool

Options:
  [--diagnose], [--no-diagnose]  # Show diagnostics (versions, configurations)

~/Devel/ChefProject/tmp$ inspec help json
Usage:
  inspec json PATH

Options:
  o, [--output=OUTPUT]                 # Save the created profile to a path
      [--controls=one two three]       # A list of controls to include. Ignore all other tests.
      [--profiles-path=PROFILES_PATH]  # Folder which contains referenced profiles.
      [--diagnose], [--no-diagnose]    # Show diagnostics (versions, configurations)

read all tests in PATH and generate a JSON summary

~/Devel/ChefProject/tmp$ inspec exec test/integration/default --format json
{"version":"0.26.0","summary":{"duration":0.037144,"example_count":3,"failure_count":2,"skip_count":0},"profiles":{"":{"supports":[],"controls":
{"(generated from test.rb:1 d5357d303b3769b9e8e88dc69924750c)":{"title":null,"desc":null,"impact":0.5,"refs":[],"tags":{},"code":"          
rule = rule_class.new(id, profile_id, {}) do\n            res = describe(*args, &block)\n          end\n","source_location":
["/opt/chefdk/embedded/lib/ruby/gems/2.1.0/gems/inspec-0.26.0/lib/inspec/profile_context.rb",181],"results":[{"status":"failed","code_desc":"File /etc/
postfix should be file","run_time":0.014447,"start_time":"2016-07-21 11:29:01 -0400","message":"expected `File /etc/postfix.file?` to return true, got false"},{"status":"failed","code_desc":"File /etc/postfix should be mode 644","run_time":0.021464,"start_time":"2016-07-21 11:29:01 -0400","message":"expected `File /etc/postfix.mode?(644)` to return true, got false"},{"status":"passed","code_desc":"File /etc/postfix should be owned by \"root\"","run_time":0.000424,"start_time":"2016-07-21 11:29:01 -0400"}]}},"groups":{"test.rb":{"title":null,"controls":["(generated from test.rb:1 d5357d303b3769b9e8e88dc69924750c)"]}},"attributes":[]}},"other_checks":[]}~/Devel/ChefProject/tmp$

~/Devel/ChefProject/tmp (master *%)$ kitchen verify
-----> Starting Kitchen (v1.10.2)
-----> Setting up <default-ubuntu-1404>...
       Finished setting up <default-ubuntu-1404> (0m0.00s).
-----> Verifying <default-ubuntu-1404>...
       Use `/Users/jmiller/Devel/ChefProject/test123/test/integration/default` for testing

Finished in 0.00027 seconds (files took 0.67817 seconds to load)

File /tmp/file
  should exist
  should be file
  should not be directory
  should not be block device
  should not be character device
  should not be pipe
  should not be socket
  should not be symlink
  should not be mounted

Finished in 0.02156 seconds (files took 0.7893 seconds to load)
9 examples, 0 failures

       Finished verifying <default-ubuntu-1404> (0m0.47s).
-----> Kitchen is finished. (0m1.88s)
~/Devel/ChefProject/test123 (master *%)$
```

### Ruby Gems, RPMs, DEBs, etc.
If you're going to operate Chef without Internet access, you will need to utilize an artifact server to host packages, binaries, gems, rpms, etc.

One very good option for this is Artifactory Pro. It supports almost any artifact / repo known to man, and it allows mirroring rubygems.org or acting like a caching proxy to it.
https://www.jfrog.com/confluence/display/RTF/RubyGems+Repositories

### ChefDK
Host the chefdk package internally so that users can easily install it.

### Chef Client
Host the chef-client package internally; to be used for bootstrapping nodes and in local development with `test-kitchen`

You can reference this location in your `.kitchen.yml` file:
```js
provisioner:
  name: chef_zero
    chef_omnibus_url: http://my.web.server/chef-pkgs/install.sh
```

The `install.sh` does not require much, it can be as simple as:
```sh
#!/bin/bash

cd /tmp/
wget http://my.web.server/chef-pkgs/chef-12.8.1-1.el7.x86_64.rpm
sudo rpm -Uvh /tmp/chef-12.8.1-1.el7.x86_64.rpm
```

### Bootstrap considerations
Use `--bootstrap-install-sh http://my.web.server/chef-pkgs/install.sh` knife option to point to the location of the installer script.

### Installing additional gems via chef `gem_package` resource
```ruby
gem_package 'train' do
  options('--no-rdoc --no-ri --no-user-install --source https://your.gem.server')
end
```

Another approach is to simply update what amounts to root's .gemrc
adding your repo and removing rubygems.org
```ruby
gembin = Chef::Util::PathHelper.join(Chef::Config.embedded_dir,'bin','gem')
execute 'set internal gem repo' do
 command "#{gembin} source -r https://rubygems.org/ -a https://your.gem.server"
 action :run
end
```

Alternatively, you can bundle up the gems you need into a tarball, host that as an artifact internally, then download and extract:

```sh
$ gem install --no-rdoc --no-ri --install-dir tmp/ --no-user-install mixlib-install
Fetching: artifactory-2.3.3.gem (100%)
Successfully installed artifactory-2.3.3
Fetching: mixlib-versioning-1.1.0.gem (100%)
Successfully installed mixlib-versioning-1.1.0
Fetching: mixlib-shellout-2.2.6.gem (100%)
Successfully installed mixlib-shellout-2.2.6
Fetching: mixlib-install-1.1.0.gem (100%)
Successfully installed mixlib-install-1.1.0
4 gems installed
$ ls -l tmp/
drwxr-xr-x  2 jmiller  staff   68 Jul 19 19:01 build_info
drwxr-xr-x  6 jmiller  staff  204 Jul 19 19:01 cache
drwxr-xr-x  2 jmiller  staff   68 Jul 19 19:01 doc
drwxr-xr-x  2 jmiller  staff   68 Jul 19 19:01 extensions
drwxr-xr-x  6 jmiller  staff  204 Jul 19 19:02 gems
drwxr-xr-x  6 jmiller  staff  204 Jul 19 19:02 specifications
$ cd tmp
$ tar cvf gems.tar *
```

Use it later in a recipe:
```ruby
download_location = ::File.join(Chef::Config[:file_cache_path], 'gems.tar')

remote_file download_location do
  source 'https://your.artifact.server/gems.tar'
  action :create
end

execute 'extract gems' do
  command "tar -xvf #{download_location} -C #{node[:languages][:ruby][:gems_dir]}"
  not_if do
    ::File.exist?("#{node[:languages][:ruby][:gems_dir]}/somegem")
  end
end
```

### Other Chef Infrastructure
One of the best methods of installing Chef components is with the `chef-ingredient` cookbook.
 * https://github.com/chef-cookbooks/chef-ingredient

Alas, it requires installation of `mixlib-install` ruby gem and all of its dependencies.  Refer to the gem installation methods above to accomplish this in an Air Gapped environment.

## Cookbook Repositories
Cookbooks inevitably have dependencies; managing those in the development phase is accomplished via Berkshelf.

### Chef Server
As of version 12.4.0, Chef Server has a Universe endpoint, which provides the same output as Supermarket or berkshelf-api universe endpoints.

```js
# Berksfile
source :chef_server

metadata
```

 If a source is configured with the location :chef_server, then Berkshelf will use the configured Chef Server as an API source. This requires Chef Server 12.4.0 or newer, or Hosted Chef.

### Internal Supermarket
A private supermarket provides an easily searchable cookbook repository (with friendly GUI and command line interfaces) that can run on a company’s internal network.

https://blog.chef.io/2015/12/31/a-supermarket-of-your-own-running-a-private-supermarket/

You can publish to a Supermarket in several ways, some notable methods:
 * https://github.com/sethvargo/stove
 * https://github.com/chef/knife-supermarket (feature has been moved into core Chef in versions greater than 12.11.18)

```js
# Berksfile
source 'https://yourinternal.supermarket.com'

metadata
```

### Minimart
MiniMart allows you to specify a list of cookbooks to mirror in YAML. MiniMart will then download any of the listed cookbooks and their dependencies to your machine. Once you are satisfied with the mirrored cookbooks, MiniMart can generate a Berkshelf compatible index of the available inventory.

 * https://electric-it.github.io/minimart/

```js
# Berksfile
source 'https://yourinternal.minimart.com'

metadata
```
