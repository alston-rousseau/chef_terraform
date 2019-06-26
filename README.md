# Chef A2 Terraform

Consists of
* Chef Automate 2 server
* Chef Server standalone or Chef HA Frontend/Backend clusters
* Route53 DNS records
* Application Load Balancer

Currently only installing the packages and doing basic configuration - the rest is left as an exercise to the reader.

## Chef Server Standalone
### Manual Config
* Set data collector token from Automate server and configure Chef Server
  * `sudo chef-server-ctl set-secret data_collector token '<API_TOKEN>'`
  * `sudo chef-server-ctl restart nginx`
  * `sudo chef-server-ctl restart opscode-erchef`
  * Edit /etc/opscode/chef-server.rb and add:
    ```
    data_collector['root_url'] = 'https://automate.example.com/data-collector/v0/'
    # Add for chef client run forwarding
    data_collector['proxy'] = true
    # Add for compliance scanning
    profiles['root_url'] = 'https://automate.example.com'
    ```
  * `sudo chef-server-ctl reconfigure`
* Create a Chef Server organisation
  * `sudo chef-server-ctl org-create test TestOrg`
  * Copy the generated validator key for configuring unattended node registration
* Create a Chef Server user and associate it with the test org
  * `sudo chef-server-ctl user-create admin Admin User admin@example.com -o test -p`
  * Copy the generated user client key for configuring knife on your workstation
* Grant the user Server Admin permissions (optional)
  * `sudo chef-server-ctl grant-server-admin-permissions admin`

## Chef HA
### Frontend Manual Config
* Visit: https://docs.chef.io/install_server_ha.html
* All FEs: `sudo cp chef-server.rb /etc/opscode/chef-server.rb`
* FE1: `sudo chef-server-ctl reconfigure`
* FE1: `scp /etc/opscode/private-chef-secrets.json ${var.ami_user}@<FE[2,3]_IP>:`
* FE1: `scp /var/opt/opscode/upgrades/migration-level ${var.ami_user}@<FE[2,3_IP>:`
* FE[2,3]: `sudo cp private-chef-secrets.json /etc/opscode/private-chef-secrets.json`
* FE[2,3]: `sudo mkdir -p /var/opt/opscode/upgrades/`
* FE[2,3]: `sudo cp migration-level /var/opt/opscode/upgrades/migration-level`
* FE[2,3]: `sudo touch /var/opt/opscode/bootstrapped`
* FE[2,3]: `sudo chef-server-ctl reconfigure`

### Backend Manual Config
* Visit: https://docs.chef.io/install_server_ha.html
* Leader (BE1): `sudo chef-backend-ctl create-cluster`
* Leader (BE1): `scp /etc/chef-backend/chef-backend-secrets.json ${var.ami_user}@<BE[2,3]_IP>:`
* Follower (BE[2,3]): `sudo chef-backend-ctl join-cluster <BE1_IP> --accept-license -s chef-backend-secrets.json -y`
* All BEs: `sudo rm chef-backend-secrets.json`
* All BEs: `sudo chef-backend-ctl status`
* For FE[1,2,3]: `sudo chef-backend-ctl gen-server-config <FE_FQDN> -f chef-server.rb.FE_NAME`
* For FE[1,2,3]: `scp chef-server.rb.FE_NAME USER@<IP_FE[1,2,3]>:`

## Backend failure recovery
https://docs.chef.io/backend_failure_recovery.html

## Validation
* Configure knife.rb
  ```
  # See http://docs.chef.io/config_rb_knife.html for more information on knife configuration options

  current_dir = File.dirname(__FILE__)
  log_level                :info
  log_location             STDOUT
  node_name                "admin"
  client_key               "#{current_dir}/admin.pem"
  chef_server_url          "https://chef.example.com/organizations/test"
  cookbook_path            ["#{current_dir}/../cookbooks"]
  ```
* Copy client key from Chef Server user-create command output to `admin.pem`
* `knife environment create production`
* `knife role create base`
* `knife environment show production`
* `knife role show base`
* Check Automate 2 Event Feed for environment & role updated events

## Unattended node registration
https://docs.chef.io/install_bootstrap.html#unattended-installs

* One option is to use AWS Systems Manager Parameter Store to make the validator securely available to nodes via IAM policy & roles
  * `aws ssm put-parameter --name "chef_validator" --type "SecureString" --overwrite --value "$(cat validator.pem)"`
* `chef_unattended_registration` includes IAM config for allowing EC2 instances access to the validator key in SSM
* `chef_clients` includes an example of unattended node registration. `chef_clients.count` variable controls how many clients are deployed.

## Alston's Notes

Sorry this is a little late and sloppy. I got distracted configuring and testing the Chef HA terraform module. I also do not have access to Roblox JIRA or cannot find my credentials?  Most of these tasks can be run simultaneously by separate engineers with the exception of the knife validation. 

Chef does not recommend multi region HA due to the need for 1.5ms latency. This code does support multi-AZ HA and it's tested working. However - the below process is repeatable and allows for multiple chef environments if node configuration latency is an issue for some reason. 

Engineers developing this system need root access to front/backend servers once deployed until configuration is complete. 

Current Chef HA module for terraform requires Route 53 domain and Certificate Manager. Might need to modify for customer but it's beautiful written terraform 11 code. It's open sourced for commercial use. 


* Determine Custom Route 53 domain to use for Chef
* Fork https://github.com/alston-rousseau/chef_terraform to FogSource
* Upgrade https://github.com/alston-rousseau/chef_terraform to terraform 12 if desired
* Remove VPC module from chef_terraform repo and configure to use standard fog source module
* Aquire Chef Automate License and save in ./ root of module
* Separate the ./chef_alb module so that the ACM or Certificate creation process for the ALB does not stall the rest of the deployment
* Codify Chef HA frontend and backend manual config. (See readme and customize to customer needs)
* If desired automate unattended node registration using AWS SSM Parameter Store 
* Knife validation using knife.rb 
