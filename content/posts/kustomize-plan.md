+++
title = "kustomize-plan"
date = "2025-07-16"
author = "keithfz"
description = "Doing Kustomize the Terraform way?"
+++

I prefer Kustomize over Helm for a few reasons. For starters, it makes it much easier to manage a large number of environments by having some shared base, and the mental model of different layers is easy for me to understand coming from things like [Yocto](https://docs.yoctoproject.org/overview-manual/yp-intro.html#the-yocto-project-layer-model) (truly a nightmare). I won't even get into the YAML templating aspects of Helm, since that's already been [discussed to death](https://ruudvanasseldonk.com/2023/01/11/the-yaml-document-from-hell#templating-yaml-is-a-terrible-terrible-idea). But Kustomize isn't fool-proof either. I have yet to find a widely adopted, easy to use tool for wrangling YAML that doesn't have a ton of foot-guns. Jsonnet looks interesting, especially something like with Grafana Tanka, but it frankly just doesn't have the same weight as Helm or Kustomize at the moment.

For Kustomize, you have to be careful your base layers don't move out from under you. You might have a change you think is safe, but it's being pulled in somewhere as a remote base, or it's a component that impacts the wrong environment. Most annoyingly of all is a change that breaks `kustomize build` so ArgoCD just errors out and never applies any updates at all until you fix your bad YAML. Ultimately, you could think a change doesn't impact something else, but how can you be sure? 

I think the [rendered manifests pattern](https://akuity.io/blog/the-rendered-manifests-pattern) does a very good job of dealing with this very same problem. It removes the abstraction, so what you see is what you get when it comes time to look at the change you're deploying via GitOps. Unfortunately, there are times where I've been working in environments that weren't easily compatible with this pattern. I wanted a quick and dirty tool to spot check deltas between branches. So inspired by `terraform plan`, I spent a couple of hours throwing together `kustomize-plan`.

It's relatively simple, but I was a bit surprised I couldn't find anything similar on GitHub at the time of writing. All it is doing is pulling down two branches, running `kustomize build` on each branch, and playing "spot the difference."

I created an example GitOps repo [here](https://github.com/keithfz/test-gitops) where I have two simple environments named dev and staging, each with their own respective kustomization files. I created a branch aptly named [new-branch](https://github.com/keithfz/test-gitops/tree/new-branch) with some changes to those kustomizations.

From the root of the local repo, all you need to do is run `kustomize-plan -f dev -f staging --new-branch new-branch --old-branch main` and the output should show the following:

```diff
Searching in folder: dev...

Resources to create:
+	apiVersion: autoscaling/v2
+	kind: HorizontalPodAutoscaler
+	metadata:
+	  name: podinfo
+	spec:
+	  maxReplicas: 4
+	  metrics:
+	  - resource:
+	      name: cpu
+	      target:
+	        averageUtilization: 99
+	        type: Utilization
+	    type: Resource
+	  minReplicas: 2
+	  scaleTargetRef:
+	    apiVersion: apps/v1
+	    kind: Deployment
+	    name: podinfo
+



Resources to delete:
-	apiVersion: v1
-	kind: Service
-	metadata:
-	  name: podinfo
-	spec:
-	  ports:
-	  - name: http
-	    port: 9898
-	    protocol: TCP
-	    targetPort: http
-	  - name: grpc
-	    port: 9999
-	    protocol: TCP
-	    targetPort: grpc
-	  selector:
-	    app: podinfo
-	  type: ClusterIP



Resources to modify:
	 ### Modified Deployment/podinfo ###
 	 apiVersion: apps/v1
 	 kind: Deployment
 	 metadata:
 	   name: podinfo
 	 spec:
 	   minReadySeconds: 3
 	   progressDeadlineSeconds: 60
 	   revisionHistoryLimit: 5
 	   selector:
 	     matchLabels:
 	       app: podinfo
 	   strategy:
 	     rollingUpdate:
 	       maxUnavailable: 0
 	     type: RollingUpdate
 	   template:
 	     metadata:
 	       annotations:
 	         prometheus.io/port: "9797"
 	         prometheus.io/scrape: "true"
 	       labels:
 	         app: podinfo
 	     spec:
 	       containers:
 	       - command:
 	         - ./podinfo
 	         - --port=9898
 	         - --port-metrics=9797
 	         - --grpc-port=9999
 	         - --grpc-service-name=podinfo
 	         - --level=info
 	         - --random-delay=false
 	         - --random-error=false
 	         env:
 	         - name: PODINFO_UI_COLOR
 	           value: '#34577c'
 	         image: ghcr.io/stefanprodan/podinfo:6.9.1
 	         imagePullPolicy: IfNotPresent
 	         livenessProbe:
 	           exec:
 	             command:
 	             - podcli
 	             - check
 	             - http
 	             - localhost:9898/healthz
 	           initialDelaySeconds: 5
 	           timeoutSeconds: 5
-	         name: newname
+	         name: podinfod
 	         ports:
 	         - containerPort: 9898
 	           name: http
 	           protocol: TCP
 	         - containerPort: 9797
 	           name: http-metrics
 	           protocol: TCP
 	         - containerPort: 9999
 	           name: grpc
 	           protocol: TCP
 	         readinessProbe:
 	           exec:
 	             command:
 	             - podcli
 	             - check
 	             - http
 	             - localhost:9898/readyz
 	           initialDelaySeconds: 5
 	           timeoutSeconds: 5
 	         resources:
 	           limits:
 	             cpu: 2000m
 	             memory: 512Mi
 	           requests:
 	             cpu: 100m
 	             memory: 64Mi
 	         volumeMounts:
 	         - mountPath: /data
 	           name: data
 	       volumes:
 	       - emptyDir: {}
 	         name: data
+


1 manifests to create, 1 manifests to delete, 1 manifests to update.

Searching in folder: staging...

Resources to modify:
	 ### Modified Service/podinfo ###
 	 apiVersion: v1
 	 kind: Service
 	 metadata:
 	   name: podinfo
 	 spec:
 	   ports:
 	   - name: http
 	     port: 9898
 	     protocol: TCP
 	     targetPort: http
 	   - name: grpc
-	     port: 19999
+	     port: 9999
 	     protocol: TCP
 	     targetPort: grpc
 	   selector:
 	     app: podinfo
 	   type: ClusterIP


0 manifests to create, 0 manifests to delete, 1 manifests to update.

TOTAL
1 manifests to create, 1 manifests to delete, 2 manifests to update.
```

So this attempts to capture any create, update, or delete operations that would occur if that PR were to get merged in.

Can try it out here: https://github.com/keithfz/kustomize-plan
