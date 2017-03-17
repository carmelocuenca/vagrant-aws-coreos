# -*- mode: ruby -*-
# # vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH	=	File.join(File.dirname(__FILE__),	"user-data")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$instance_name_prefix = "core"

if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|
  config.vm.box = "dummy"
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.provider :aws do |aws, override|
    # The access key for accessing AWS
    # aws.access_key_id =

    # The region to start the instance in, such as "us-east-1"
    aws.region = "us-east-1a"

    # The AMI id to boot, such as "ami-12345678"
    aws.ami = "ami-eff12ff9" # core2
    aws.instance_type = "t2.micro"
    aws.elastic_ip = true
    # An array of security groups for the instance. If this instance will be
    # launched in VPC, this must be a list of security group Name. For a
    # nondefault VPC, you must use security group IDs instead
    aws.security_groups = %w(sg-365fa149)
    # The subnet to boot the instance into, for VPC.
    aws.subnet_id = "subnet-96b6cee0" # vagrantVPC publicSubnet
    aws.keypair_name = "ec2privatekey"
    override.ssh.username = "core"
    override.ssh.private_key_path = ENV['HOME'] + "/.aws/awskey.pem"
    # always use Vagrants insecure key
    override.ssh.insert_key = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    # Multiple machines are defined within the same project Vagrantfile using
    # the config.vm.define method call
    config.vm.define vm_name = "%s-%02d" % [$instance_name_prefix, i] do |config|
      config.vm.hostname = vm_name

      # To automatically replace the discovery token on 'vagrant up'
      if File.exists?(CLOUD_CONFIG_PATH) && ARGV[0].eql?('up')
        user_data_specific	=	"#{CLOUD_CONFIG_PATH}-#{i}"

        require 'yaml'
        data = YAML.load(IO.readlines(CLOUD_CONFIG_PATH)[1..-1].join)
        yaml = YAML.dump(data)

        File.open(user_data_specific,	'w') do |file|
          file.write("#cloud-config\n\n#{yaml}")
        end

        config.vm.provider :aws do |aws, override|
          aws.private_ip_address = "10.0.0.#{100+i}"
          aws.user_data = File.read(user_data_specific)
        end
      end
    end
  end
end
