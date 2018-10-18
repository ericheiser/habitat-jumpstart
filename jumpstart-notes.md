## Setup:
https://github.com/chef-cft/habitat-workshop-environment
cd habitat-workshop-environment
ssh-add -l
okta_aws solutions-architects
bundle exec carpenter build GM_HAB_WRKSHP (Choose 3: solutions-architects)

If you want to get rid of that build:
bundle exec carpenter destroy GM_HAB_WRKSHP

Base Code: https://github.com/chef-training/habitat_jumpstart
Slides: https://docs.google.com/presentation/d/1UAhAV0Cj9MelkIODMzamL0408SogT8NAqA99cbphhBY/edit#slide=id.g39841ec8ca_4_0



# Start of Workshop commands

$ ssh hab@<YOUR IP ADDRESS>   password: chefhab1!
$ cd ~
$ touch <your first_AND_last name>

# Install Habitat
$ curl https://raw.githubusercontent.com/habitat-sh/habitat/master/components/hab/install.sh | sudo bash
$ hab --version

# Mod 2: Builder account setup
GitHub accounts are integrated into Builder.   
If you don’t already have a GitHub account, FIRST create one at https://github.com

Sign into Habitat Builder:   https://bldr.habitat.sh/#/sign-in 
Allow it to authorize the permissions to GitHub

Select “Create Origin” in Builder
    Input your desired Origin Name. 
    Recommended Origin Name:  Use your shortname or fullname.  No Spaces!
    Select “Public Packages”
    Save 

Select your Profile (upper right hand corner in Builder) 
Select “Generate Token”
Copy generated token for use on next slide

Setup Habitat CLI
$ hab setup
    Set up a default origin? [Yes/no/quit] Yes
    Default origin name: [default: chef] <your origin>
    Create an origin key for `<your origin>'? [Yes/no/quit] Yes (get from cat ~/.hab/etc/cli.toml)
    Set up a default Habitat personal access token? [Yes/no/quit] Yes
    Habitat personal access token: <paste your personal access token here>
    Set up a default Habitat Supervisor CtlGateway secret? [Yes/no/quit] no
    Enable analytics? [Yes/no/quit] no

Navigate to https://github.com/chef-training/habitat_jumpstart and fork the repository to your personal account                  
Clone *YOUR* habitat_jumpstart GitHub repo 

# Mod 3: Node.js app
$ cd habitat_jumpstart/project_nodejs
$ hab plan init
$ tree habitat
$ cat habitat/plan.sh
$ hab studio enter
    [default:/src:0]# ls
    [default:/src:0]# build
    [default:/src:0]# source results/last_build.env
    [default:/src:0]# hab svc load $pkg_ident
    [default:/src:0]# hab pkg install core/curl -b
    [default:/src:0]# curl http://localhost:8000  

GO TO →
http://YOUR_WORKSTATION_IP:8000
To see the supervisor output in Hab studio:
`sup-log` or 'sl' (Press Cntl+C to exit)

Unload (i.e stop) the Hab services you are running in Hab Studio
[default:/src:0]# hab svc status   
[default:/src:0]# hab svc unload <PACKAGENAME>   

# Mod 4: Project - Node.js + Docker
[default:/src:0]# source results/last_build.env
[default:/src:0]# hab pkg export docker results/$pkg_artifact 
[default:/src:0]# exit 

$ sudo docker image ls
$ source results/last_docker_export.env
$ sudo docker run -it -p 8000:8000 $name

GO TO →
http://YOUR_WORKSTATION_IP:8000  (Ctrl+C to exit)

# Mod 4.2: Section 4: National Parks Application
$ cd ~/habitat_jumpstart/national-parks   
$ hab plan init 
$ nano habitat/plan.sh 
    (use https://gist.github.com/kkwentus/c23718a5b3ac98ee3438ae6093423020#file-1-np-plan-sh-v1)
    After copying contents into your plan.sh, set pkg_origin to the name of *your* Builder Origin

$ nano habitat/hooks/init 
    Content:   NP hooks init v1 https://gist.github.com/kkwentus/c23718a5b3ac98ee3438ae6093423020#file-1-np
$ nano habitat/hooks/run (same link)
$ nano habitat/default.toml  

~/habitat_jumpstart/national-parks  
$ hab studio enter
    [1][default:/src:0]# build
    [1][default:/src:0]# source results/last_build.env
    [1][default:/src:0]# hab pkg export docker results/$pkg_artifact
    [1][default:/src:0]# exit
$ source results/last_docker_export.env
$ sudo docker run -it -p 8080:8080 $name

GO TO →  http://YOUR_WORKSTATION_IP:8080/national-parks/

# Mod 5: Binding

$ nano habitat/plan.sh
    Update plan.sh  https://gist.github.com/kkwentus/c23718a5b3ac98ee3438ae6093423020#file-5-national-parks-plan-sh-v2-with-bindings
    National Parks plan.sh v2 with bindings

Add content to your init hooks file:  init hook v2 with bindings
Add content:  run hook v2 with bindings

Update Content: habitat/default.toml  
    mongodb_database = "demo"  (add to the top of the file)

$ nano habitat/mongo.toml

~/habitat_jumpstart/national-parks  
$ hab studio enter
    [default:/src:0]# build
    [default:/src:0]# source results/last_build.env
    [default:/src:0]# hab svc load core/mongodb/3.2.10/20171101045425
    [default:/src:0]# hab config apply mongodb.default $(date +%s) habitat/mongo.toml
    [default:/src:0]# hab svc load $pkg_ident --bind database:mongodb.default

Clean-up
    [default:/src:0]# hab svc status
    [default:/src:0]# hab svc unload scottford/national-parks
    [default:/src:0]# hab svc unload core/mongodb
    [default:/src:0]# exit

# Section 6: Binding across supervisors
$ nano habitat/haproxy.toml 
    https://gist.github.com/kkwentus/c23718a5b3ac98ee3438ae6093423020#file-9-haproxy-toml

$ hab studio enter  (MUST use this version of mongodb)
    [default:/src:0]# build
    [default:/src:0]# source results/last_build.env
    [default:/src:0]# hab pkg export docker core/mongodb/3.2.10/20171101045425
    [default:/src:0]# hab pkg export docker core/haproxy
    [default:/src:0]# hab pkg export docker results/$pkg_artifact
    [default:/src:0]# exit 

$ source results/last_build.env
$ source results/last_docker_export.env
$ sudo hab sup run &
$ sudo docker run -d core/mongodb --peer <PRIVATE_WORKSTATION_IP> 
$ sudo hab config apply mongodb.default $(date +%s) habitat/mongo.toml

Start 3 containers running Tomcat app (outside Studio)
Open 3 new SSH consoles / windows and repeat the following in each:
    $ cd habitat_jumpstart/national-parks/
    $ source results/last_docker_export.env
    $ mongo_id=$(sudo docker ps -qf "ancestor=core/mongodb")
    $ mongo_ip=$(sudo docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $mongo_id)
    $ sudo docker run -it $name --bind database:mongodb.default --peer <PRIVATE_WORKSTATION_IP> --strategy rolling --topology leader
      (^^ use -d instead of -it to run in the background) (use 'ohai cloud' to get private IP)

Start and configure haproxy (outside of Studio)
Open one more SSH session and execute the following:
    $ cd habitat_jumpstart/national-parks/
    $ source results/last_docker_export.env
    $ sudo docker run -d -p 8085:8085 -p 8000:8000 core/haproxy --bind backend:national-parks.default --peer <PRIVATE_WORKSTATION_IP>
    $ sudo hab config apply haproxy.default $(date +%s) habitat/haproxy.toml


Point your browser to http://<public ip>:8085/national-parks/ and you should now see the app balanced across 3 containers!

Now browse to http://<public ip>:8000/haproxy-stats and use “admin” and “password” as username and password. This shows the app containers binding with haproxy.

# Section 7: Deploy new changes to your locally running Hab app

Update your running National Parks Hab app
    Change the pkg_version in your plan.sh to any version greater than what it is currently set to.  (i.e 6.3.2)
    Update the <h1> title in src/main/webapp/index.html
    Do a local build 
    Promote the package into the Stable channel

$ export HAB_AUTH_TOKEN=<Your Builder Personal Access Token>
$ hab studio enter
    [default:/src:0]# build
    [default:/src:0]# source results/last_build.env
    [default:/src:0]# hab pkg upload results/$pkg_artifact
    [default:/src:0]# hab pkg promote $pkg_ident stable
