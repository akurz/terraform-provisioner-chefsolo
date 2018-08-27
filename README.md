Terraform Provisioner Chef Solo
==================

Usage
---------------------

Chef solo provisioner allows you to run chef with local mode but still being able to discover other nodes provisioned by chef.

It is designed to interact closely with the chef solo data-source.

Required attributes for using chefsolo provisioner are :

`chef_module_path` : The path to the module / cookbook / policy you want to run on your machine.

`output_dir` : Where all the generated files by the provisioner should be output

`nodes` : An array of Node JSON files, chef will use them in order to discover nodes during provisioning
( these can be very easily generated by the chefsolo datasource, refer to this : https://github.com/Mwea/terraform-provider-chefsolo/ )

`target_node`: A DNA JSON file, this is the DNA that will be applied to your machine. ( same as above, this file can be generated automatically with this plugin : https://github.com/Mwea/terraform-provider-chefsolo/ ) 

`instance_id`: The node ID you want to target during chef run (must match with the targeted node)

A very minimal configuration would be :

```hcl
provisioner "chefsolo" {
    "chef_module_path": "path/to/my/cookbook"
    "output_dir": "/tmp/output_tf"
    "nodes": ["${data.template_chefsolo.nodes.*.node}"]
    "target_node": "${data.template_chefsolo.nodes.*.dna}"
    "instance_id": "my_id"
}
```

Other options will soon be documented too.

Example of usage with terraform provider chef solo : 

```hcl

data "template_chefsolo" "consul_server" {

  count = "${length(var.instances_ips)}"

  automatic_attributes = "${file("${path.module}/attributes/master.tpl")}"
  node_id         = "${element(var.instances_hostnames, count.index)}"
  policy_name     = "consul_server"
  policy_group    = "local"
  environment     = "preprod"

  vars {
    node_ip     = "${element(var.instances_ips, count.index)}"
    node_id     = "${element(var.instances_hostnames, count.index)}"
    hosts       = "${jsonencode(zipmap(
                        var.instances_ips,
                        var.instances_hostnames)
                    )}"
  }
}

resource "null_resource" "provision_consul" {

  triggers {
    always = "${uuid()}"
  }

  count = "${length(var.instances_ips)}"

  connection {
    user        = "centos"
    private_key = "${file(var.ssh_key)}"
    host        = "${element(var.instances_ips, count.index)}"
  }

  provisioner "chefsolo" {

    use_sudo         = true
    nodes            = ["${data.template_chefsolo.consul_server.*.node}"]
    target_node      = "${element(data.template_chefsolo.consul_server.*.dna, count.index)}"
    environment      = "${element(data.template_chefsolo.consul_server.*.environment, count.index)}"
    use_policyfile   = "${element(data.template_chefsolo.consul_server.*.use_policyfile, count.index)}"
    instance_id      = "${element(data.template_chefsolo.consul_server.*.node_id, count.index)}"

    chef_module_path = "${path.module}/../.."
    output_dir       = "${path.module}/bump-chef"
    version          = "12.18.31"

  }
}
```
