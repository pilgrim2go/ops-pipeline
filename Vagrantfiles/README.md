# Adding a new use case into izanamee

Lets walk through adding a new use case on top of izanamee.  We'll start from the [`izanamee/headless`](https://atlas.hashicorp.com/izanamee/boxes/headless) base and create a jenkins build server, a sonar server, and add a job to build a project that will be available as soon as we boot up.

## decide which base to use
[`izanamee/headless`](https://atlas.hashicorp.com/izanamee/boxes/headless) or [`izanamee/desktop`](https://atlas.hashicorp.com/izanamee/boxes/desktop) will be the best choices for base boxes.

We'll start from headless, but the process is the same for starting on desktop.

## pull in any new cookbooks
We'll use the [sonarqube cookbook](https://supermarket.chef.io/cookbooks/sonarqube) from chef supermarket to provision sonarqube for us.

Add the depend line for sonarqube into the Berksfile.

    cookbook 'sonarqube',          '~> 0.3.2'

## create a new roll to combine the parts you need
Create a new [chef role](https://docs.chef.io/roles.html) in `provision/chef/roles/sonar.json`.  You can base it off of the `izanamee-headless.json` role in that same dir.  Just add the following to the `run_list in the new role file

    "run_list": [
      "role[izanamee-headless]",
      "recipe[packer-boss-jenkins]",
      "recipe[sonarqube]",
      "recipe[jenkins-job]"
    ]

# make a Vagrantfile to raise and provision
Create a vagrant file that ups `izanamee/headless` in the root of the repo named `Vagrantfile`.  You can use [`Vagrantfiles/jenkins-headless/Vagrantfile`](jenkins-headless/Vagrantfile) as a base for your new use case.  We'll just end up doing slightly different things in the provisioning section.

`jenkins-job` is a cookbook that allows taking a `config.xml` as used by a jenkins job, and loads it into jenkins.

Add the role into the `Vagrantfile` provisioning section, and any extra attribute settings you need.  The one here will add the test_sonar.xml file into jenkins as the job named `test_sonar`.

    ...
    config.berkshelf.enabled = true
    config.vm.provision 'chef_solo' do |chef|
        chef.json = {
          'jenkins-job' => {
            'job-files' => {
              'test_sonar' => 'test_sonar.xml'
            }
          }
        }
        chef.roles_path = 'provision/chef/roles'
        chef.data_bags_path = 'provision/chef/data_bags'
        chef.add_role 'sonar'
    end
    ...

Now, when you `vagrant up`, `izanamee/headless` will be download if needed and booted up.  The sonar role will be provisioned against that box.

# create a new packer build
If you need this use case to be built and packaged as boxes for future users, you can add a new packer build under the `packer` directory.

You can start by basing your new build on the `packer/headless.json` build.  Just add the same recipes or roles from above into the new build json.

    {
      "type": "chef-solo",
      "cookbook_paths": [
        "provision/chef/cookbooks",
        "provision/chef/vendor-cookbooks"
      ],
      "data_bags_path": "provision/chef/data_bags",
      "roles_path": "provision/chef/roles",
      "run_list": [
        "role[izanamee-headless]",
        "recipe[packer-boss-jenkins]",
        "recipe[sonarqube]",
        "recipe[jenkins-job]"
      ]
    },