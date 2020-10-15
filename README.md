# DRAFT: This is the opinions of Chris Matteson, and not official guidance or endorsed by HashiCorp


# Terraform Code Style Guide

Like all languages, Terraform is infinitely flexible in how it can be used. Very different code can result in the same deployment. This document attempts to provide opinionated guidelines, which while optional for Terraform to work, have been collected from countless hours of working with the language by dozens of people in the real world scenarios.

This guide should be used as a preferred approach. It does not attempt to accommodate all edge cases, and will leave some questions unanswered. It’s not intended to be adhered to to the detriment of all other goals, but understood that largely following this philosophy will result in more reusable, and easier to understand code.


## Definitions:

For the purpose of clarity, this document offers the following definitions:

**Terraform Configuration File: **A file ending in .tf or .tf.json which contains Terraform code

**Top-Level Keyword:** The only allowable words to begin a top level block in Terraform

**Block:** A container for other content and usually representative of the configuration of some kind of object, like a resource. Blocks have a block type, can have zero or more labels, and have a body that contains any number of arguments and nested blocks. Most of Terraform's features are controlled by top-level blocks in a configuration file.

**Argument:** Assign a value to a name. They appear within blocks.

**Expression:** Represent a value, either literally or by referencing and combining other values. They appear as values for arguments, or within other expressions

**Module:** A collection of .tf or, .tf.json or .tfvars files kept together in a directory and intended to be managed in version control.  This can also include sub folders for artifacts to be parsed by the terraform code, and artifacts of version control or workflow such as *.ci, .gitignore and *.md files

**Root Module: **The root module is built from the configuration files in the current working directory when Terraform is run, and this module may reference child modules in other directories or remotely sourced, which can in turn reference other modules, etc.

**Child Module:** A module which isn’t typically run directly from the Terraform CLI, but instead consumed by another module. This module creates a “component”, by completing it’s own unique function.

**Generalized Child Module:** A special type of child module intended for very broad consumption. Well written Terraform Public Registry modules are generally Generalized Child Modules. Unlike most child modules because they often include significantly more  inputs/outputs and switching logic in the code to allow them to solve most or all possible use cases for the component.


## Core Paradigm

Terraform at its core is a human readable, machine parsable, declarative immutable language for defining API addressable infrastructure. The approach possesses an incredible amount of power, as well as flexibility. As part of the defined state approach, Terraform creates a Directed Acyclic Graph (DAG) to define all of the relationships between resources. This allows for multithreading, while simplifying the code. However it also renders the order of the code meaningless to the machine parsing the code. Another unusual factor of the language is that the basic container of code is not a function, or even a file, but a non-recursive directory. Resources can be placed in any terraform file in the directory without altering how Terraform will function.

This guide addresses foremost the human readability of the code. Even though lots of code will achieve the same results, the ability to understand this code is not equivalent. Humans tend to consume information in a semi linear fashion like a book. Books have several features in common with Terraform code:



*   Books are meant to be consumed by the (human) reader in a certain way
*   Books have an intended audience, and the author makes assumptions about what that audience knows
*   Short books might exist without further structure, but long books are broken into chapters
*   Information which needs to be looked up should is cataloged as to be easy to find in an index

While this analogy isn’t perfect, the net result is usually good Terraform code, furthermore it’s also helpful for resolving a lot of decisions when writing code that could go either way by other standards.


## Module Structure

This section covers the structure of files inside a Terraform module, be that a root module or child module.


### The Invisible Edge Problem

This concern is broken out above all others because it represents the most problematic common practice of Terraform code, which inherently undermines the human readability of Terraform code because of two basic features of Terraform. As mentioned in the prior section, Terraform:



1. Creates a DAG, so the order of code is irrelevant
2. Blindly combines all Terraform files in a directory

The result is the invisible edge problem which undermines the readability of the code. By separating resources, data sources, etc into separate terraform files inside a directory, we create edges across those files while are completely invisible unless we read and understand all of the code in every file. Each individual file cannot be evaluated by itself, because dependencies could exist which won’t be discoverable within that file.

There are three exceptions to this which don’t impact the ability to understand the Terraform code, and none of them include resources, data sources, or providers.



1. Inputs: These cannot be dependent on anything else, and will also clearly represent just one thing. Putting them in their own file makes it easier to understand what levers exist to modify when using the module.
2. Outputs: The opposite of Inputs, these are at the other end of the code, and represent only information which should be extracted from the module. Again useful for reference, and thus are better exposed in their own file.
3. Terraform statements: Backend configuration, required_versions, required_providers, and experiments parameters. These all are independent of the Terraform code itself but takes action on how Terraform interacts with the code, and thus should be independent. 

The right solution to this problem is Submodules. Submodules provide very clear edges using variables and outputs. It's completely reasonable to read just a module block and understand enough about how that submodule is used without needing to actually read the rest of the module code.

In anything except the most simple demo code which is being written, submodules will provide an easier to understand and more scalable approach. practice this tends to reduce readability.


### Terraform File Names

As mentioned, Terraform will combine all files ending in .tf or .tf.json within a single directory. What those files are named before the extension is purely up to the user. That flexibility has resulted in a lot of confusion, and demand for prescription. In nearly all cases, the following are all the files which should exist, although not all of them need to exist if they wouldn’t have any content.

**main.tf:** Contains all data, local, provider, and resource blocks

**outputs.tf:** Contains all output blocks in alphabetical order

**terraform.tf:** Contains a single terraform block which defines backend, required_version, required_providers, and experiments parameters.

**variables.tf:** Contains all variable blocks in alphabetical order

It’s still very common to a lot of other terraform file names used. The following bullets attempt to cover the logic behind these choices vs other options:

**Other Locations for Variables/Outputs:** Because they are the contract of communication terms between humans and the module or between modules, Variables and Outputs are reference materials and should be easy to lookup. Placing them in their own files makes them quick and efficient to look up.

**data.tf: **Separating out data sources into its own file. Data blocks are intimately tied to Resource blocks. They can both be depended on and depend on Resources. Unlike Variables or Outputs, they don’t provide any significant value to the human reader to have them in reference format since they aren’t actionable. They are best included in the main.tf.

**backend.tf: **Separating out the backend configuration into its own file. This generally results in a very short file, like all files potentially including Terraform blocks. Thus they are better combined into a single file.

**versions.tf: **Same logic as backend.tf. There isn’t enough in any files with terraform blocks to justify having its own file.

**override.tf:** A special terraform file to [override other resource definitions](https://www.terraform.io/docs/configuration/override.html). No need to use this in the typical workflow.

**providers.tf:** Separating out providers into its own file. While not recommended, providers can also be dependent resources. Similar to data sources, these aren’t needed as reference. They should exist where they make sense in the code, usually at the top of the main.tf where they set the stage which the data sources and resources will tell.

**&lt;component>.tf:** It’s very common to still see individual components or resource types separated into their own files. The issue here is addressed in the “Invisible Edge Problem” above. 


### Other Files in Module

Multiple other files are often included within Terraform Modules because they are used by or relevant to the work Terraform does. Many of these should be in subfolders. The guidance is to use the following structure



*   **.gitignore:** A file for those using a git based version control with Terraform code. At minimum it should look like this:

    ```
    #  Local .terraform directories
    **/.terraform/*

    # .tfstate files
    *.tfstate
    *.tfstate.*

    # .tfvars files
    *.tfvars
    ```


*   **README.md:** Readme file in markdown. Should provide at minimum an introduction to the module and its purpose. For Generic Child Modules, should be much more verbose with usage examples, and explanation of at least primary choices when utilizing.
*   **/files:** Directory which holds static files which Terraform utilizes. If the files are being copied somewhere else, they already be named the same with the same extension as they will be at their destination.
*   **/templates: **Directory which holds dynamic template files to be used by the template data source or function. Should be named the same, including extension as they will at their destination, except with an additional extension of .tmpl added.
*   **/modules: **Directory which holds submodules.
*   **/examples:** Directory for child modules to show working examples of how to use the module.
*   **/tests:** Directory to contain any tests for the Terraform module.


#### What not to include in module

Not everything should be in the module, even if it could be.



*   **Binaries: **Recommend using an Artifact Repository, such as Artifactory. Because Terraform typically goes into VCS, and binaries fit poorly in VCS
*   **Unrelated Files:** Files which don’t directly relate to Terraform shouldn’t be included.


### One module for everything, or multiple modules using External State?

The ultimate bounds of root modules can often be difficult to determine. The reality is that the decision here is typically better to be based on business and organizational realities than what can be done with Terraform. Terraform code should have clear owners and lifecycles, if different parts cross between parts of the organization to perform related but different tasks, like creating underlying networking vs compute, it’s often easier to have separate modules.

Another related and common issue is when one set of Terraform code stands up infrastructure which is used by another provider, such as standing up EKS, then using the helm provider to deploy a workload. While the Terraform DAG will properly order the resources across the provider boundary, Terraform refresh does not understand the provider boundary, and if something were to occur after deployment, such as the EKS cluster being deleted, all applies will fail because a refresh can’t be performed on the helm deployment. In general, providers should exist at or next to the beginning of the DAG when possible.


### Should this be a module?

One of the most common questions people have is determining if a segment of code should be a module. The primary considerations for this should be:



*   Consider child modules to be “chapters” of the book. Use chapters to make longer books easier to follow. This can effectively replace the use of component files, although it might not look exactly the same.
*   Module “chapters” should have their own story arc. The module shouldn’t exist to:
    *   Wrap a single resource (or just a couple resources).
    *   Wrap a single type of resources like IAM which are used for a bunch of unrelated resources.
*   The general rule is that the code to call a child module should replace at least 5x the lines of code it would take to perform the work.
*   It’s perfectly acceptable for the root module to:
    *   Do nothing but call child modules
    *   Have no child modules (if short)
    *   Have a mix of resources and child modules


### Where should this module be called from?

There are multiple places where a module can live. Because Terraform is Infrastructure as Code, the recommendation is to always have Terraform Modules saved into version control such as git. Where the module lives depends on its target audience and lifecycle.



*   **Submodule:** These modules exist inside a modules folder of another module. Because of this they will share the same git repository. The benefits of this pattern are that it avoids double commits that would be necessary to properly update child modules in a git repository or registry, and there is no need to run terraform init -upgrade. However submodules generally aren’t usable externally. They are best for child modules which exist purely to simplify the root module by breaking apart an otherwise large main.tf.
*   **Public Git Repository: **These modules are public, but aren’t easily discoverable. It’s a great place to store work which doesn’t contain secret or proprietary information. Public git repositories allow modules to be discovered and reused as submodules in other projects or by other people. Unlike the registry, they don’t support version negotation, but there also isn’t a bar in expected completeness or reusability.
*   **Private Git Repository: **Similar advantages to public git repositories, except that it could potentially include proprietary information. Secrets are still generally discouraged.
*   **Public Registry:** The Terraform Public Registry at ([https://registry.terraform.io](https://registry.terraform.io)). Modules here will also be backed by a git repository as well, but can be called using the registry source syntax which allows for version negotiation to allow easier tracking of dependent modules.
*   **Private Registry: **Similar advantages to Public Registry, but since it’s unique to an organization, it’s much easier to write modules which cover all of the potential use cases for an organization, without having to solve a problem in a truly generic fashion. It’s common that organizations will have best practices on how to implement certain behaviors and Private Registry modules are the perfect place to implement these policies.


### Generalized Child Modules

There exists a special type of child module used for managing a discrete component with the flexibility to handle most or all potential permutations via variable inputs. The verified [terraform-aws-modules/vpc](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws) module on the registry is a great example. These modules can be so fundamental that there isn’t a need to review the child module code when trying to understand the root module. This is akin to a book which mentions the protagonist using a car. The book isn’t about cars, and doesn’t try to explain what a car is. It’s assumed the reader knows. If the reader wants to know more, they can investigate further and find books that describe cars in great detail.

By trying to accomodate so many use cases, these modules also often violate a few goals of this guide. Namely they can be incredibly complex with a ton of switching logic that means most of the code isn’t ever used.


### Git Structure

There is more nuance to deciding where a module (which isn't in a modules subfolder of another module) lives besides just being in a public or private git. The easiest concept is for modules to tie one to one with  Git repositories. However this isn’t always the case in practice.



*   **One-to-One:** Each module lives in Git, within the root folder of the git repository. This is the default choice for most modules.
    *   Advantages:
        *   Simple to understand
        *   Permissions can be granularly controlled
    *   Disadvantages:
        *   Discovery can be difficult
        *   Leads to repository sprawl
*   **Monorepo: **A single repository for all Terraform code. 
    *   Advantages:
        *   One place to find all Terraform code
    *   Disadvantages:
        *   Limited control of permissions
        *   Difficult to test or use in CI pipeline as unrelated changes can trigger events
*   **Subfolder in Application Code Repository:** A module in a ‘terraform’ folder for a related non Terraform project.
    *   Advantages:
        *   Easy to discover
        *   Easy to use in CI pipeline which triggers deployments based on changes to project code without changing Terraform code
    *   Disadvantages:
        *   May be mistaken as “official” deployment method, even if it isn’t
        *   Not applicable to a lot of situations


### Code Formatting

The Terraform parser allows you some flexibility in how you lay out the elements in your configuration files, but the Terraform language also has some idiomatic style conventions which we recommend users always follow for consistency between files and modules written by different teams. Automatic source code formatting tools such as “terraform fmt” may apply these conventions automatically.



*   Indent two spaces for each nesting level.
*   Any non-empty, multiline set of curly braces or square braces should end with a closing brace on its own line.
*   When multiple arguments with single-line values appear on consecutive lines at the same nesting level, align their equals signs:

        ```
        ami           = "abc123"
        instance_type = "t2.micro"
        ```


*   When both arguments and blocks appear together inside a block body, place all of the arguments together at the top and then place nested blocks below them. Use one blank line to separate the arguments from the blocks.
*   Use empty lines to separate logical groups of arguments within a block.
*   For blocks that contain both arguments and "meta-arguments" (as defined by the Terraform language semantics), list meta-arguments first and separate them from other arguments with one blank line. Place meta-argument blocks last and separate them from other blocks with one blank line.

    ```
    resource "aws_instance" "example" {

      		  count = 2 # meta-argument first

      		  ami           = "abc123"
      		  instance_type = "t2.micro"

          network_interface {
        		    # ...
      		  }

      		  lifecycle { # meta-argument block last
        		    create_before_destroy = true
      		  }
        }

    ```



*   Top-level blocks should always be separated from one another by one blank line. Nested blocks should also be separated by blank lines, except when grouping together related blocks of the same type (like multiple provisioner blocks in a resource).
*   Avoid separating multiple blocks of the same type with other blocks of a different type, unless the block types are defined by semantics to form a family. (For example: root_block_device, ebs_block_device and ephemeral_block_device on aws_instance form a family of block types describing AWS block devices, and can therefore be grouped together and mixed.)


### Reusability

Well written Terraform modules should be reusable without changes. Like printing multiple copies of a book, multiple people should be able to simultaneously use the module. Terraform itself doesn’t have a problem with this, but many cloud resources require unique names. There are three well used mechanisms to prevent these types of naming collisions:



*   **Random Resource:** Use the random resource, either random_id, or random_string to create a chunk of text that can be inserted into name parameters to make them unique. This is the preferred method.
*   **Prefix Variable: **Use a required variable to allow the user to specify a unique string to be inserted into name parameters.
*   **Both:** Have an optional prefix variable, and use logic to create a local which is the random resource string if the prefix variable is empty.


### Ordering

Ordering inside the file should be relatively linear. While Terraform is multithreaded, humans usually are not. The best order is likely the same order a person might have gone through all the steps were they planned out. General best practices:



*   Dependent resources are defined in the file before those that depend on them
*   Providers at the top of the file to “set the scene”
*   Globally used items at the top of the file after providers:
    *   Randomly generated strings used throughout the file for naming
    *   Globally useful data sources like azurerm_client_config
    *   Locals
*   Terraform code files should logically “build”. While we don’t enforce ordering, the file should be ordered.
*   Similar resources should be grouped if it doesn’t disrupt the logic. I.E. group a bunch of IAM policies for the same resources together, but if your code builds two distinct things, the IAM policies for each of those shouldn’t be merged into a single IAM section


### Comments

Comments are crucial for understanding complicated code. They however can also become distracting when adding more verbosity than necessary. It’s crucial to consider the audience when considering comments. I typically recommend writing code oriented towards someone else with a solid but not necessarily complete mastery of Terraform. Terraform is already “documentation” so the recommended guidelines are:



*   Consider the audience when writing comments. Typically the recommendation is writing code oriented towards someone else with a solid but not necessarily complete mastery of Terraform. Do not write to the lowest common denominator
*   Comments should add additional detail, often the “why”, not restate something which the resource clearly demonstrates.
*   Be consistent with types of comment delimiters used. General preference by populatority:
    *   Use hash (#) exclusively over double slash (//)
    *   Prefer hash (#) to multiline comments (/* */) unless the comment is very long, such as or for use while commenting out resources
*   Metadata often requires comments
    *   Metadata including lifecycle or count usually represent a lot of additional information regarding their purpose, but which typically can’t be understood from reaching just the singular resource block. Add a quick comment that addresses why this is used.
*   Loops, counts, and other complicated dynamic code almost always deserves a comment.
*   It should never take longer than a minute from someone adept in Terraform, but not familiar with a certain codebase to understand what a particular resource or data source is doing.


### Dynamic Code

Terraform provides a lot of ways to minimize the code that needs to be written, one of the most powerful mechanisms is the ability to write dynamic code. Terraform 0.12 introduced some new constructs such as for_each and dynamic blocks to supplement count, and 0.13 introduced further dynamic features with the extension of count and for_each to modules.

Shorter, more dynamic code however is also more complex, and the readability of the module can be lost in the mission to save lines. Do use dynamic code, but as mentioned in the comments section, almost any use of dynamic code merits at least a one line explanation of what is going on.

One special consideration is using Count as an on/off switch. It’s a common practice dating back to at least a blog from 2016. It’s an incredibly powerful mechanism to make the code applicable to more use cases. However it’s very easy to abuse. For modules which are not “Generalized”, **it’s usually better to use the copy->paste->edit workflow to create a new module than to use count as a switch.**


### Provider and Module Versions

To prevent external changes from causing unintentional changes, it’s highly recommended that providers and modules specify versions which they are tied to. Depending on the level of acceptable risk and management effort to be tracking version updates, that can either be hard locked to a particular version, or use looser mechanisms such as less than next major rev, or using tilde to track through bug fix versions.

One significant recommendation is to **never use the version statement within a provider block**. Instead a required_providers statement should be used inside a terraform block. This results in more predictable behavior when using child modules.


## Secrets in Terraform Code

Due to the nature of configuring infrastructure, there are almost always secrets which Terraform is or could be exposed to. Terraform's CLI considers the entire state to be a sensitive artifact, assuming the user will keep it secure, and so doesn't do anything special beyond hiding values in the CLI output. Terraform Enterprise and Terraform Cloud improves on this process via encryption of the state file through HashiCorp Vault. However there are still several other considerations:



*   Secrets should be passed to providers via provider specific environment variables whenever possible. This is the only way to keep these secrets out of the state file. While Terraform variables will keep them out of the code, those variables will be loaded into the state file.
*   When used in a pipeline within a CI tool or Terraform Enterprise, there is still a risk that a malicious Terraform code committer could use local_exec provisioner or external data source to use “printenv” to access the credentials. Terraform Enterprise should always include a Sentinel policy to prevent the use of the local_exec provisioner or external data source from being used by default. Those, as well as third party providers should be treated as items which need to be whitelisted per workspace.
*   Another option is to use the Terraform Vault provider to query dynamic cloud credentials for each run. The problem of this method is determining how to authenticate. The easiest method is to provide each workspace with its own token, but this is obviously a lot of manual effort. Instead Vault Agent could be used to authenticate the entire server, but there would be no way to provide different cloud permissions per workspace. Additionally Terraform code needs to be inserted for Terraform to know to make this call. Finally Terraform providers don’t have a way to signal to Vault that the run has finished, meaning that the dynamic credentials need to live longer than the maximum expected run, but in many instances someone with access to the state file could retrieve the credentials before they expire. Likewise Terraform doesn’t have a way to refresh the credential so defining too short of a validity period could result in a long run failing.


## Application Onboarding

Organizations which deploy underlying shared Terraform or Terraform Enterprise infrastructure but don’t work to build an onboarding process risk making the tool shelfware. A documented onboarding process should be created and followed for each application which is migrated into Terraform. All of this should be designed to make the usage of Terraform easier, as the more often people are pushing code, the more confidence they will have in it. Each of the following sections should be considered when designing this process.


### People

It’s a disservice to the project to not consider the people who are the stakeholders when onboarding an application. Obviously some people will be creators of Terraform code, but others will be consumers. They may have a more limited understanding of Terraform or all the nuance that goes into how certain things should be configured at the organization. 

Additionally people are often divided into different organizations which sometimes split the responsibilities for infrastructure. Terraform code for an application will need to acknowledge these divisions. Sometimes separating code for things such as networking from code from compute. While the preference is to follow current recommendations like those from AWS Terraform Landing Zones to separate all workloads into their own accounts, the organizational reality doesn’t always allow for that, and separating application workspaces to mirror organizational borders can ease adoption.


### Process

Existing processes need to be acknowledged in the process of Terraforming an application. Ideally those processes should be codified via Terraform modules which implement the allowed behavior and Sentinel policies which enforce that behavior. If the consumers of the infrastructure which Terraform is deploying are not technical, Self Service options such as ServiceNow integration with Terraform Enterprise allows for more of this process to be automated.


### Terraform Enterprise Workspace Onboarding Process

Due to limitations with securing cloud credentials which is discussed in greater depth in the “Secrets in Terraform Code” section, it becomes necessary in most situations for Terraform Enterprise Workspaces to be provisioned and configured by a central IT team or TFE owners. Part of that process should include provisioning credentials on a per team, or ideally per workspace basis, and furthermore establish a process for updating those credentials on a regular basis. One potential option is to use the TFE and Vault providers to manage workspaces with code and update the credentials by running Terraform regularly.


## Testing

Some of the greatest value of Infrastructure as Code workflows comes from the ability to perform automated testing. There are numerous kinds of testing available for Terraform of increasing complexity. One important consideration is to **NOT test that Terraform itself works**. That means test the logic of of your code is acceptable but don’t test that the aws_vpc resource actually creates a VPC in AWS. We already have those tests in the provider.



*   Syntax Checking
    *   terraform fmt -check - Validates syntax. Suggest to use as precommit hook.
    *   terraform validate - Validates HCL. Suggest to use as a precommit hook.
*   Linting
    *   tflint - Validate that valid options are selected. Now supports 0.12 and is being updated to use plugins to allow support beyond AWS.
*   Lifecycle test
    *   terraform plan, terraform apply, terraform destroy - Validates that code will actually apply and destroy successfully. Use for Pull Requests or nightlies depending on overhead
*   Unit test
    *   Sentinel - HashiCorp tool for writing policies. Able to ingest Terraform plan and configuration to evaluate arbitrary rules. Technically designed for policy as code and not unit testing.
    *   Conftest - Platform from Open-Policy for testing configuration files. Less Terraform aware than Sentinel.
*   Pipecleaning test
    *   Test the application stack which Terraform is deploying to ensure it’s up. This usually gives a good indication that the Terraform code worked. These tests can be in any language/platform as they aren’t actually testing Terraform. This doesn’t need to be the full suite of tests that are run against the application, just enough to validate the infrastructure deployed correctly. One preferred practice is for the application to have some health endpoint which can be queried to validate the application, a simple curl to that endpoint would then be a sufficient test.
*   Functional Testing
    *   Terratest and Kitchen Terraform - two platforms for doing functional testing to ensure resources really did get deployed. This should be used to test logic in the Terraform code, but it’s not advised to test that every resource is deployed as expected. Terraform already has tests to validate that resources work in each provider, there is no reason to implement this logic.

It’s rare that organizations move immediately to end to end testing. Instead testing is often an iterative process. The recommendation is to implement easy tests such as linting and syntax checks, then implement high value but easier to build tests such as pipecleaning tests. Unit testing and functional testing should be sprinkled in last where they are not redundant.


## Policy as Code

Sentinel provides a powerful new mechanism to accelerate change cycles by automating process review which previously would have needed to be performed manually.

When writing Sentinel Policies, it’s important to keep in mind that all policies should do one of two things:



*   Control Costs
*   Enforce Behavior

The recommendation is to at minimum have a policy which restricts usage of the local_exec provisioner, external data source, and custom providers. This is to prevent malicious committers gaining access to credentials. Individual workspaces should be whitelisted as necessary to use these.

The Terraform Foundational Policies Library includes a ton of great examples on how to use Policy as Code with Terraform:

[https://github.com/hashicorp/terraform-foundational-policies-library](https://github.com/hashicorp/terraform-foundational-policies-library)


## Lifecycle

As the usage of Terraform matures, the lifecycle of changes begins to take on increasing importance. The first thing to note is that the lifecycle of the Application, and the lifecycle of the Terraform code which deploys it, often are different, and can be treated differently. While application changes are typically a frequent event which can be promoted through a long lived branching strategy, Terraform changes are usually much less frequent after the initial deployment, and different environments should use identical code with only the variables being different. Thus it usually doesn’t make sense to use long lived branches with Terraform code.

With Terraform Enterprise or Terraform Cloud, it’s recommended that the VCS workflow be used with any non-development environment. To minimize the number of code pushes, Terraform development should use remote execution using the remote backend with the Terraform CLI. The “prefix” option is preferred over the “name” option when configuring the remote backend to allow the code to be reused across environments without modification.


## Workload Provisioning

Terraform is excellent at deploying infrastructure. Sometimes that also includes deploying an workload onto the infrastructure. Workloads differ from infrastructure in that they often have their own code and require the user to handle upgrades vs letting it be handled by the cloud provider. This means the Terraform user needs to consider how to best handle the deployments and upgrades. There are numerous different approaches to each piece:



*   Philosophical
    *   **Config Management/Deployment Tools vs Terraform:** Terraform focuses on API addressable resources. Config Management tools such as Puppet, Chef, or Ansible can work with API addressable resources as well, but their sweet spot tends to be managing configuration inside an operating system. Furthermore most config management solutions have methodologies for handling drift and run on a periodic basis so updates can be made. Additionally there are specialized deployment tools like XLDeploy and Webdeploy that manage inside the OS configuration for select applications.
    *   **Scheduler vs Terraform:** Modern methodology for managing workload deployment using a tool such as Kubernetes or HashiCorp Nomad. Schedulers typically support blue/green and canary deployments along with integrated health checks.
*   Method to Configure a VM (in order of preference)
    *   **Immutable Images:** Using a tool such as HashiCorp Packer, immutable images can be created automatically in a pipeline then stored in an artifact repository. This method requires up front investment in process, but provides the greatest degree of consistency by ensuring that every deployment is identical.
    *   **Userdata: **All the major cloud providers support Cloud Init to pass configuration information to a VM. This can be in the form of a shell script, or something more complicated such as calling a config management tool. These typically only run once, and can be extremely brittle due to timeouts or unexpected changes in external resources being called. Terraform does not validate if the userdata script is successful.
    *   **Provisioners:** This is considered an anti-pattern and should be avoided unless no other option is available. Provisioners allow Terraform to execute arbitrary commands but present a security issue by necessitating access to the resources from wherever Terraform is running, and generally handle idempotency, failures, and drift poorly.
*   Upgrade Process for Workload
    *   **Deploy New Image:** Either immutable or configured via userdata/provisioners. A new image can be deployed and configuration adjusted to utilize it. Terraform natively does not handle this cutover gracefully. By default it will destroy the old workload first, resulting in downtime. Lifecycle metadata can be used to configure create before destroy, minimizing downtown, but Terraform still has no concept of health checks or session draining.
    *   **Rerun Provisioners:** If provisioners are being used, they could be tainted and reran. The success or failure however falls on the code themselves and isn’t well handled by Terraform.
    *   **Config Management/Deployment Tools:** Have other tooling handle this outside of Terraform’s knowledge. Terraform can potentially “kick” the tool the first time, and let it handle updates in the future.
    *   **Scheduler: **This is the preferred methodology. Schedulers can handle lifecycle, while understanding health checks and roll backs. Terraform should be used to configure all the underlying infrastructure then pass off workload management to a scheduler on top of that infrastructure.
