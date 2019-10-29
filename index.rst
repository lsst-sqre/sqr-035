..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

   Configuration management and deployment infrastructure for Kubernetes-based services for the LSST Science Platform and SQuaRE Services. 
   
.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).


Context
=======

This document does three things: lays out the elements and practice we use for kubernetes-based services, outlines guidelines for best practices, and a discussion of current and upcoming technological choices for implementation.

SQuaRE engineers maintain 20+ services and 30+ deployments of those services in production at this time in a wide range of enviroments (commodity, LDF, single-host). They lie in the entire range of the maturity range.

While SQuaRE follows the DevOps model of operating the services it develops, there is still a concept of an operator - on-call rotation, vacation backups etc are an example of someone who is in charge of a service in whos development they were not ivolved.

Aside from good engineering discipline, the great number of SQuaRE services pushes us to a unified approach in this area; on the other hand, we also believe in the right tool for the job. The interplay between these two principles has led to the architecture described in this technote. 
   
Elements
========

The elements of service deployment are:

Container Release
  The container(s) with the service to be deployed. 

Configuration Management
  What is in the deployment (eg. contingress, certs)

Secrets
  Credentials required by the deployment

Deployment Orchestration
  How the deployment happens 

Configuration Control
  When the deployment happens

During earlier stages of maturity, development of each one of those steps can be progressively delivered. For example a container may be manually built; a helm chart can be manually applied; etc. However for complete configuration control and operational maintainability, the entire chain needs  to be in place.

In technical terms, as service maturity evolves, configuration management moves from imperative to declarative. 

Container Release
-----------------

For Kubernetes model, the quantum of deployment is a Docker image. 

Containers by themselves are the end result of the release management process and they themselves are a reflection of possibly different approaches to that (cadence, testing, floating v. pinned dependencies).

In certain models (nublado) there is additional environmental injection to the container. This means that the injection layer in and of itself is part of the service. 

Configuration Management
------------------------

The principle here is fairly simple: deployment = service + configuration

The two Kubernetes native ways of doing this are Helm and Kustomize. Kustomize is simple and attractive for lightweight services; Helm has the advantage of support templating via Tiller and being a popular source of 3rd party charts (eg Influx).

At SQuaRE we stalled for a long time on this debate, as different developers have (strong) preferences in one direction or the other. We eventually realise this was a false dichotomy and our desire to standardise was being applied at the wrong layer (configuration management v. deployment orchestration).

Recommendation: Use either Helm or Kustomize (or even both - an emerging model is Helm + "last mile" Kustomize) according to the needs of the service and the prefernace of the developer. See the section on Deployment Orchestration for dicussion. 

The other big debate in this are is whether to store configuration with the code of the service or in a deployment repository full of just the configurations of related services (or even many deployments of the one service).

Recommendation: store deployment configuration in its own repository (eg. lsp_deploy). This makes it far easier to find if you are not familiar with the service, easier to provide GitOps support (see below) and easier to modify for different deployments.

Secrets
-------

Secrets should always be curated so that they are securely stored and retrievable by the ops team.

Recommendation: use Vault

Note this should be done as early as possible given it is compatible with both imperative and declarative models. 

.. code-block:: bash

				vault kv list secret/dm/square/scipipe-publish/${ENV_NAME}/



Deployment Orchestration
------------------------

Our debates in this area centered as to whether to use terraform or a more kubernetes-native system for orchestrating the deployment. For a long time this was the Terraform v. Helm debate. Terraform was seen as having the advantage of being able to sequence complex deployments of services consisting of multiple containers, and being in use for non-Kubernetes deployments (eg. virtual machine-based architectures). On the other hand Helm was seen as much easier to understand and a native match to Kubernetes applications. It can be argued that sequencing (launch A, when it is up launch B becaue B requires A, etc) is unecessary for well-designed Kubernetes services that provide health points (launch A and B together, A monitors B's health endpoint and goes green when it detects B is available). 

We recently had a prototype cycle with ArgoCD, and we are coming to the conclusion that using a native Kubernetes continuous deployment service such as ArgoCD resolves both the Helm v. Kustomize debate as well as the Helm v. Terraform debate. ArgoCD features remove the need to template Helm with Tiller and have easy support for "last mile" Helm+customize models. Moreover it provides a unified interface for executing GitOps configuration control irrespective of the underlying configuration management used. While we prefer unsequenced Kubernetes services, ArgoCD does have support for "wave" deployment that means terraform is not required even for sequenced deployments. 

Recommendation: Use ArgoCD to orchestrate deployment of Helm, Kustomize and Helm-Kustomize services.

In addition, products like ArgoCD provide clear deployment dashboards that allow an operator to assess the health of a system and verify configuration control. 


Configuration Control
---------------------

Configuration Control is an outcome that can be achieved in a number of ways, ranging from process-driven ways (formal change control, compliance) to SRE-driven ways (automation, continuous deployment infrastructures etc). 

Recommendation: use GitOps (automated deployment by a system driven from  a git merge to master or other special branch) as it is suited to both models (if regulatory gatekeeping is required, it can be performed before merge is authorized).

The compelling advantage of GitOps is that it exposes a layer understood by all developers (git) which allows an operator to perform core maintainance operations (rolling back to a previous known-to-be-good version or doing a security patch for a dependency) without an underlying knowledge of the deployment architecture (eg Helm, kustomize or whatever else).

This is also the best supported model in deployment infrastructure products. 


Deployment add-ons
====================

While not strictly speaking involved in the deployment process, monitoring and logging should be part of service deployment.

We would like to also have service auto-discovery though our ideas for implementing this across all services are not full formed yet. 


   
.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
