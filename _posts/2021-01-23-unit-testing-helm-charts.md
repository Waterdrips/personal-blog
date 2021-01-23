---
layout: post
title: How to unit-test your helm charts with Golang
description: We explore unit testing Helm charts with Golang
date: 23-01-2021
image: helm.jpg
headline_img: helm.jpg
image_alt: OpenFaaS Logo
length: 8
---

Learn how to write Golang unit tests for your Helm charts to keep quality high and make changes with confidence.

In both my open source and commercial work I have needed to write [helm charts](https://helm.sh/docs/topics/charts/). 
These charts are a definition of how to install an application on kubernetes. They often include configuration options
like if we should enable auth, clustering or specific database options. 

These options usually boil down to some "logic" or code within the helm chart. Here is an example of 
a configuration option being used to set a flag.

```yaml
{% raw %}
{{- if .Values.auth.enabled }}
  - --enable-auth
{{- end }}
{% endraw %}
```

This is a basic example and often these logic statements can be much more complex and have larger impacts to many different
parts of the chart. 

## How do we test this? 

The software developer in me wants to wtie tests for this. If I pass the `auth.enabled=true` flag when calling helm install 
I want to be sure that my above logic generates the expected outcome. 

I did a lot of searching before embarking on my own solution for testing this type of logic at a unit test level. 
Most of the options I found were for an integration level test, such as the official [helm website](https://helm.sh/docs/topics/chart_tests/).
This level of testing is great, but can be slow to test out the potentially hundreds of options and flags we may want to 
set in our chart. 

So, I set out in search of a way to define my unit tests in code. I wanted to define what I expected the helm command to 
generate and then compare what was actually generated.

## The code

I chose `Golang` - partly because I use golang every day, and partly because `Helm` is written in go. This means we can 
potentially import parts of the helm codebase into our test library and avoid "re inventing the wheel". 

The first place to look is the code for the `helm` cli binary its'self, we want to be able to leverage the existing implementation
so our tests match our real world use.

This snippet adapted the `helm apply` command is a good place to start:

```go

func Template(name, namespace, chartPath, filePath string, valueFilePaths, overrideValues []string, output interface{}) error  {
    // Some setup code omitted for readability
    ...

    // This is where the magic happens, This takes the values and the template and runs the helm logic
	release, err := client.Run(chart, values)
	if err != nil {
		return err
	}

    // A function to split the output of the template, (A single array of all files in 1 variable)
    // it returns a map of filename to  contents
	manifests := splitChart(release.Manifest)

	if _, exists := manifests[filePath]; !exists {
		return fmt.Errorf("no file found at path%s", filePath)
	}
    
    // convert the returned YAML to JSON, all internal kubernetes communication uses JSON
	jsonBytes, err := yaml.YAMLToJSON(manifests[filePath])
	if err != nil {
		return err
	}
    // Unmarshal the file (as json now) into a struct, We will see how we use k8s definitions in the test to reduce 
    // code even more!
	if err = json.Unmarshal(jsonBytes, &output); err != nil {
		return err
	}

	return nil
}

``` 

This allows us to write tests where we define the struct we expect our file to match, and then compare that expected value with what
we get. Below is an example using a simple ingress record.

```go 
func Test_IngressWithTLS(t *testing.T) {
	want := v1.Ingress{
		TypeMeta:   metav1.TypeMeta{
			Kind:       "Ingress",
			APIVersion: "networking.k8s.io/v1beta1",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:                       "helmChart-nginx-example",
			Labels:                   map[string]string{"app.kubernetes.io/instance": "helmChart", "app.kubernetes.io/name": "nginx-example"} ,
		},
		Spec:       v1.IngressSpec{
			Rules: []v1.IngressRule{
				{Host: "chart-example.local" , IngressRuleValue: v1.IngressRuleValue{HTTP: &v1.HTTPIngressRuleValue{Paths: nil}}},
			},
		},
	}
	var got v1.Ingress

	if err := Template("helmChart",
		"default",
		"./chart",
		"template/ingress.yaml",
		[]string{"./chart/values.yaml"},
		nil,
		&got,
	); err != nil {
		t.Fatalf("got an error templating chart: %v", err)
	}

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("difference in chart and expected. Got:\n%+v\nWant:\n%+v\n", got, want)
	}
}

```

We can now describe what we expect our chart to produce in our code, we can add logic to generate sections, like if 
we enabled TLS in the values with `--set` then we could add a section in our `want` struct to match what the helm template logic
should generate.

When we make changes to our chart we can then use our tests to validate that we dont make any other unintended changes. For example
we could update an image version in our base `values.yaml`, run our tests and see that our `deployment` now fails because we 
didn't update out tests to use the new image.

### The Upsides
With a good suite of unit tests written we can be confident that when we make changes to our chart that they have the 
desired effect. Its hard having to manually verify the changes each time and this is time we could better spend developing 
more features! 

This type of testing is particularly helpful for new members of the team, they can make changes with confidence and start having 
an immediate impact. Additionally, when reviewing helm chart changes we get a version of the change written in a more familiar
 and strongly typed language. This should reduce the time taken to digest the changes and make informed decisions when 
 reviewing your team's PRs.


### The downsides?

It wouldnt be fair to not talk about a few possible downsides when showing off this technique. Using this approach 
(using go code from helm) means vendoring large chunks of the k8s.io repos and pulling in helm. This 
is a lot of dependencies we have just added to our codebase. The other alternative is to write a go wrapper that executes
the `helm template` binary with the desired values, this removed the need to vendor k8s/helm code. However, it does mean 
we would need to write our own version of the `v1.Ingress` structs. I have chosen to vendor the code because it makes 
writing tests quicker as we don't need to keep adding to our own type definitions of the kubernetes objects.


## Full example
I have written a full example with a basic helm chart [in this repo](https://github.com/Waterdrips/helmunit)

This is my second attempt at unit testing Helm charts. I took a similar approach when writing the [OpenFaaS Cloud helm chart](https://github.com/openfaas/openfaas-cloud/tree/master/chart/test).
Last time I went with the "DIY" approach and wrote the type definitions and a wrapper around the helm binary. You can check out
the implementation at the link.


Let me know what you think [on Twitter](https://twitter.com/alistair_hey) as I would be keen to hear if everyone else is 
unit testing their helm charts or if you see the value in this approach.