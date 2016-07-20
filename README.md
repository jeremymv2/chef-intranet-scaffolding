# Chef Intranet Scaffolding
What kind of scaffolding does it take to run Chef on a network with no Internet access?

### Busser / Serverspec
Historically, the trickiest part of operating Chef in an Air Gapped environment has been local development when using the `test-kitchen` verifier.
The default for a long time has been busser / serverspec (this changed to `inspec` in version 0.16.28 of `test-kitchen`)

The trouble is that Busser is not Proxy friendly and does not allow use of an alternate GemSource to https://rubygems.org
 * https://github.com/test-kitchen/busser/issues/21
 * https://github.com/test-kitchen/busser/issues/33

The recommended approach is simply migrating over to `inspec` rather then try to tackle setting proxy environment variables, creating custom box images with the gems pre-installed, etc.

`inspec` does not reach out to the Internet for any additional gems to install. Additionally, migrating from `serverspec` is pretty straight forward. Keep in mind that `inspec` currently does not allow for nested `describe` blocks, other than that, it's resources are mostly the same with extra functionality and flexibility of custom extensions.

 * https://docs.chef.io/inspec_reference.html
 * https://github.com/chef/inspec

### Ruby Gems, RPMs, DEBs, etc.
If you're going to operate Chef without Internet access, you will need to utilize an artifact server to host packages, binaries, gems, rpms, etc.

One very good option for this is Artifactory Pro. It supports almost any artifact / repo known to man, and it allows mirroring rubygems.org or acting like a caching proxy to it.
https://www.jfrog.com/confluence/display/RTF/RubyGems+Repositories

### ChefDK
Host the chefdk package internally so that users can easily install it.

### Chef Client
Host the chef-client package internally; to be used for bootstrapping nodes and in local development with `test-kitchen`

You can reference this location in your `.kitchen.yml` file:
```
provisioner:
  name: chef_zero
    chef_omnibus_url: http://my.web.server/chef-pkgs/install.sh
```

The `install.sh` does not require much, it can be as simple as:
```
#!/bin/bash

cd /tmp/
wget http://my.web.server/chef-pkgs/chef-12.8.1-1.el7.x86_64.rpm
sudo rpm -Uvh /tmp/chef-12.8.1-1.el7.x86_64.rpm
```

### Bootstrap considerations
Use `--bootstrap-install-sh http://my.web.server/chef-pkgs/install.sh` knife option to point to the location of the installer script.

### Installing additional gems via chef `gem_package` resource
```
gem_package 'train' do
  options('--no-rdoc --no-ri --no-user-install --source https://your.gem.server')
end
```

Alteratively, you can bundle up the gems you need into a tarball, host that as an artifact internally, then download and extract:

```
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
```
download_location = ::File.join(Chef::Config[:file_cache_path], 'gems.tar')

remote_file download_location do
  source 'https://s3-us-west-2.amazonaws.com/jmiller-gems/gems.tar'
  action :create
end

execute 'extract gems' do
  command "tar -xvf #{download_location} -C #{node[:languages][:ruby][:gems_dir]}"
  not_if do
    ::File.exist?("#{node[:languages][:ruby][:gems_dir]}/somegem")
  end
end
```

## Cookbook Repositories
Cookbooks inevitably have dependencies; managing those in the development phase is accomplished via Berkshelf.

### Chef Server
As of version 12.4.0, Chef Server has a Universe endpoint, which provides the same output as Supermarket or berkshelf-api universe endpoints.

```
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

```
# Berksfile
source 'https://yourinternal.supermarket.com'

metadata
```

### Minimart
MiniMart allows you to specify a list of cookbooks to mirror in YAML. MiniMart will then download any of the listed cookbooks and their dependencies to your machine. Once you are satisfied with the mirrored cookbooks, MiniMart can generate a Berkshelf compatible index of the available inventory.

 * https://electric-it.github.io/minimart/

```
# Berksfile
source 'https://yourinternal.minimart.com'

metadata
```
