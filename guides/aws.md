---
layout: page
title: "Amazon Web Services: Getting Started Guide"
permalink: /guides/aws/
---

This page helps you get started with `jclouds` API with Amazon Web Services

1. Sign up for [Amazon Web Services Signup](https://aws-portal.amazon.com/gp/aws/developer/registration/index.html).
2. Get your Access Key ID and Secret Access Key by going to this [page](https://aws-portal.amazon.com/gp/aws/developer/account/index.html?ie=UTF8&action=access-key).
3. Ensure you are using a recent JDK 6 version.
4. Setup your project to include aws-ec2 and aws-s3.
	* Get the dependencies `org.jclouds.provider/aws-ec2` and `org.jclouds.provider/aws-s3` using jclouds [Installation](/start/install).
5. Start coding

## Using S3

{% highlight java %}
import static org.jclouds.aws.s3.options.PutObjectOptions.Builder.withAcl;

// get a context with amazon that offers the portable BlobStore API
BlobStoreContext context = ContextBuilder.newBuilder("aws-s3")
                 .credentials(accesskeyid, secretkey)
                 .buildView(BlobStoreContext.class);

// create a container in the default location
BlobStore blobStore = context.getBlobStore();
blobStore.createContainerInLocation(null, bucket);

// add blob
Blob blob = blobStore.newBlob("test");
blob.setPayload("test data");
blobStore.putBlob(bucket, blob);

// when you need access to s3-specific features,
// use the provider-specific context
AWSS3Client s3Client =
     AWSS3Client.class.cast(context.getProviderSpecificContext().getApi());

// make the object world readable
String publicReadWriteObjectKey = "public-read-write-acl";
S3Object object = s3Client.newS3Object();

object.getMetadata().setKey(publicReadWriteObjectKey);
object.setPayload("hello world");
s3Client.putObject(bucket, object, withAcl(CannedAccessPolicy.PUBLIC_READ));

context.close();
{% endhighlight %}

## Using EC2

Required Maven dependencies:
<dependency>
	<groupId>org.apache.jclouds</groupId>
	<artifactId>jclouds-all</artifactId>
	<version>1.8.0</version>
</dependency>
<dependency>
	<groupId>org.apache.jclouds.driver</groupId>
	<artifactId>jclouds-log4j</artifactId>
	<version>1.8.0</version>
</dependency>
<dependency>
	<groupId>org.apache.jclouds.driver</groupId>
	<artifactId>jclouds-sshj</artifactId>
	<version>1.8.0</version>
</dependency>
* Note - make sure that the dependency is 'JAR'

{% highlight java %}

import java.util.Set;

import org.jclouds.ContextBuilder;
import org.jclouds.aws.ec2.AWSEC2Api;
import org.jclouds.aws.ec2.compute.AWSEC2TemplateOptions;
import org.jclouds.compute.ComputeServiceContext;
import org.jclouds.compute.domain.Image;
import org.jclouds.compute.domain.NodeMetadata;
import org.jclouds.compute.domain.OsFamily;
import org.jclouds.compute.domain.Template;
import org.jclouds.domain.Location;
import org.jclouds.logging.log4j.config.Log4JLoggingModule;
import org.jclouds.sshj.config.SshjSshClientModule;

import com.google.common.collect.ImmutableSet;
import com.google.common.collect.Iterables;
import com.google.inject.Module;

// get a context with ec2 that offers the portable ComputeService API
ComputeServiceContext context = ContextBuilder.newBuilder("aws-ec2")
                      .credentials(accesskeyid, secretkey)
                      .modules(ImmutableSet.<Module> of(new Log4JLoggingModule(),
                                                        new SshjSshClientModule()))
                      .buildView(ComputeServiceContext.class);

// here's an example of the portable api
Set<? extends Location> locations =
        context.getComputeService().listAssignableLocations();

Set<? extends Image> images = context.getComputeService().listImages();

// pick the highest version of the RightScale CentOS template
Template template = context.getComputeService().templateBuilder().osFamily(OsFamily.CENTOS).build();

// specify your own groups which already have the correct rules applied
template.getOptions().as(AWSEC2TemplateOptions.class).securityGroups(group1);

// specify your own keypair for use in creating nodes
template.getOptions().as(AWSEC2TemplateOptions.class).keyPair(keyPair);

// run a couple nodes accessible via group
Set<? extends NodeMetadata> nodes = context.getComputeService().createNodesInGroup("webserver", 2, template);

// when you need access to very ec2-specific features, use the provider-specific context
AWSEC2Api ec2Client = context.unwrapApi(AWSEC2Api.class);

// ex. to get an ip and associate it with a node
NodeMetadata node = Iterables.get(nodes, 0);

ElasticIPAddressApi elasticIpApi = ec2Client.getElasticIPAddressApi().get();
String ip = elasticIpApi.allocateAddressInRegion(node.getLocation().getId());
elasticIpApi.associateAddressInRegion(node.getLocation().getId(),ip, node.getProviderId());

context.close();
{% endhighlight %}


