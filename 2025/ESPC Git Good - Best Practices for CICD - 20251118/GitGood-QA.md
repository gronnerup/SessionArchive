# Git Good - Q&A Follow-Up

Thanks again to everyone who joined my **Git Good: Best Practices for
CI/CD and Collaboration in Microsoft Fabric** session!

I really appreciate the engagement and the great questions in the chat - and also your patience when I ran a bit over time 🙏

As promised, here is the full Q&A from the session.

------------------------------------------------------------------------

## Q&A

### Question:
Would you sometimes split out prepare, model and present even further
per area or data product like `prepare_sales_dev` to ensure you can keep
better capacity scaling per data product?

### Answer:
Not by default. I would always start small. That said, there are
situations where you may need to split the layers further.

For storage, this could be due to governance policies where each source
must be stored separately in isolated landing lakehouses. For the
prepare layer, it could be due to completely separate data platforms
(e.g., using the same ingestion mechanism but different preparation
units).

However, I would say that **80% of all solutions (if not more)** can be
covered by a simple layered separation. In most cases I see the model
layer being split based on **business unit or domain**. The same
typically goes for reports.

------------------------------------------------------------------------

### Question:
Also curious to the question of Allan. Mostly because we see that up
until the semantic model layer we mostly see a central team working,
while after that it splits up into multiple domains typically.

### Answer:
Exactly. And it very often also involves different personas. Data
engineers work on ingestion and data preparation, while data analysts
work on the semantic models and reports. Beyond that, report development
typically splits further across areas or departments, each working on
their own set of semantic models.

------------------------------------------------------------------------

### Question:
How can we implement source control for Lakehouse SQL views or other
objects like security and schema in Microsoft Fabric, following best
practices?

### Answer:
For lakehouses, there is currently **no built-in way** of deploying SQL
endpoint structures (like in a SQL project). Tables and files are also
**not tracked or versioned in Git**.

So, as far as I know, you must handle this **customly**. There are a few
possible approaches:

1.  **Manage views inside your transformation process (recommended).**
    For example, imagine you have a dimension table `dim_calendar`
    prepared in a notebook that refreshes daily. As part of that
    notebook, include a cell that generates the corresponding view
    (e.g., in a `dim` schema such as `dim.Calendar`). You can even fully
    automate this using:
    -   a default view pattern,
    -   metadata-driven view creation (on/off flags),
    -   a shared helper function that either generates default views or
        creates custom SQL-based views.
2.  **Use system views to extract and recreate objects.** Another
    approach is to use a utility notebook to fetch schemas, views, etc.,
    from system views (`INFORMATION_SCHEMA.VIEWS`,
    `INFORMATION_SCHEMA.SCHEMATA`, ...). Then recreate them in the
    target environment. **Be careful**, views may reference tables
    that don't yet exist. That's why I prefer to handle view creation
    **inside the notebook logic** responsible for the table itself.

------------------------------------------------------------------------

### Question:
I like the recipe approach. Is there a reason why you use this over
Terraform? Any comparisons that you've made?

### Answer:
Thanks! 🙂

My starting point was created when there **was no Terraform provider for
Fabric**. I built wrappers around the REST APIs to handle many different
scenarios, including long-running operations, throttling, etc. When the
Fabric CLI was introduced, migrating was quite easy.

Today it's more a mix of **policy and skillset**.

If the question is specifically *"why Fabric CLI + Python scripts over
Terraform?"*, then:

-   **Fabric evolves faster than Terraform**, meaning REST + CLI always
    exposes new features first.
-   The Fabric CLI allows calling **any REST endpoint**, including
    preview and even undocumented ones (with obvious risks).
-   Item-level operations are much better supported in a **purpose-built
    tool** like the CLI than via Terraform.

That said, Terraform is still a great choice for **infrastructure
outside Fabric**, such as resource groups, capacities, Key Vault, and
private networking.

And as a side note, if you're into AI and want to use **MCP
servers**, a CLI is also tailored towards that.

------------------------------------------------------------------------

### Question:
I have two dev, two UAT, two pre-prod, and one prod environment. Since a
workspace can only be attached to one deployment pipeline, I cannot
attach prod to multiple pipelines or sync changes between dev
workspaces. How should I handle this?

### Answer:
This is difficult to answer without knowing why your environments are
structured that way. If you're referring to **Fabric Deployment
Pipelines**, then I recommend looking into a more scalable,
enterprise-ready deployment pattern using **Azure DevOps pipelines or
GitHub Actions**, deploying via the **fabric-cicd** library.

`fabric-cicd` enables highly flexible and complex deployment patterns
using a **configuration-based approach**.

You can learn more here: https://microsoft.github.io/fabric-cicd/latest/

------------------------------------------------------------------------

### Question:
Are there scenarios where you would NOT use this approach? Where it
would be too big/complex? Or do you see this working for any size of
project/company?

### Answer:
The short answer is **no** - or at least not if the solution is intended
to become an ongoing, scalable production solution. I would personally
use it even for small solutions.

The exception is **POCs or experiments**. In those cases, I just do
**ClickOps**. 🙂

------------------------------------------------------------------------

### Question:
When multiple developers are working on different features
simultaneously, is it advisable to create dedicated developer workspaces
(like sandboxes) for feature development, mirroring dev?

### Answer:
I would not create dedicated **developer** workspaces, but instead
create **feature** workspaces. This also enables team members to co-work
on a feature.

------------------------------------------------------------------------

### Question:
Can you get feature workspaces back quickly, for example if a bug is
discovered?

### Answer:
This relates to restoring deleted workspaces (and branches) after a
feature is closed and merged into main.

Technically, yes. You *can* restore deleted workspaces. But I do
**not** consider that best practice.

If a feature has been closed, approved, and merged into main, I would
always create a **new feature/bugfix branch + workspace** for the fix.

It also depends on your branching strategy. In my session I mentioned
three scenarios. In scenario 2 (cherry-picking) and scenario 3
(octopus merge), you almost never need to restore workspaces because
nothing moves into main (and into production) before it is fully tested
and signed off.

------------------------------------------------------------------------

### Question:
How do you handle deploying connection IDs between environments?

### Answer:
`fabric-cicd` is your lifesaver here.

Using **parameterization** with dynamic replacement of item IDs, you can
completely control dependency references, such as switching
connection IDs between development, test, and production.

In my current GitHub repo I don't use dynamic replacements but instead
generate the `parameter.yml` file automatically for each deployment,
ensuring all workspace IDs, connection IDs, etc., are correctly mapped
across environments.

------------------------------------------------------------------------

### Question:
Is it possible to duplicate data of selected Lakehouses/Warehouses when
creating a feature workspace?

### Answer:
I would generally *not* recommend having feature- or developer-specific
storage layers. This is a tricky subject in any data platform - not
just Fabric.

Fabric's mix of Lakehouses, Warehouses, and shortcuts makes this even
more complex.

To answer the question: **Yes, it is technically possible to clone
data**, but it requires custom implementation and comes with challenges
around retention, object selection, and cost/consumption.

For Warehouses, you can create **zero-copy table clones**. For
Lakehouses, you can use **shortcuts**, or clone data manually if needed.

------------------------------------------------------------------------

### Question:
How does the database metadata deployment/release process work if you're
using metadata-driven ETL (e.g., based on Azure SQL DB/Fabric DB)?

### Answer:
If your metadata should reside in a relational store, I would always
choose a **Fabric SQL Database** for a Fabric solution.

Using Azure SQL DB: - adds extra cost, - introduces unnecessary
complexity, - often requires VNET/network security policies, - may
require managed private endpoints for Fabric to communicate with Azure
SQL DB.

Note: trial capacities are limited to **3 Fabric SQL Databases**, but
there is no limit in paid capacities.

I am also exploring a **file-based metadata** approach using Data
Contracts (YAML), which offers better version-control support - but
this is still experimental.

------------------------------------------------------------------------

### Question:
How to set up the dynamic connection for a semantic model so the model
in DEV refers to the DEV lakehouse, and the model in TEST/PROD refers to
their respective lakehouses?

### Answer:
This is handled by **fabric-cicd** during deployments to test and
production.

In development, your semantic models use a development connection. The
ID of this connection can be added manually, or as in my solution, 
added automatically together with the corresponding IDs for test and
production.

During deployment, the semantic model definition is updated with the
connection ID of the target environment.

Another option is using **Tabular Editor 2 CLI** to update the
connection during deployment.

I will be hosting a **Tabular Editor webinar on December 10th**, focused
specifically on CI/CD for semantic models (with some overlap with my Git
Good session). 
  
You can sign up here:
https://tabulareditor.com/resources/upcoming-and-on-demand-events/ci-cd-for-microsoft-fabric-and-power-bi
