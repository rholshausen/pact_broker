I would like to do a thought experiment to see if we can come up with a better design for the Pact Broker.
For instance, if we had to start from scratch, what would we keep, what would we change and what would we throw away.

We did this for the Pact framework, and it worked well. We created an RFC proposal documented at https://github.com/pact-foundation/roadmap/pull/146 and a prototype to validate the thinking. The plan for the prototype is /development/Projects/Pact/pact-janus/Documentation/project-plan.md

To start off we can look at addressing all the issues with the current Pact broker, which are:

1. The Pact Broker data model is built for Pact and is not easy to extend. PactFlow wraps it, and added BDCT by adding new tables and API paths
which are separate. The Janus prototype will require the provider shaped published and this would also require separate tables and API paths. That is documented in /development/Projects/Pact/pact-janus/Documentation/broker-integration-notes.md

2. The consumer and provider roles are hardcoded into the model. Every contract has to define a consumer and a provider, and this gets very
confusing for people when they start adding message interactions and have to state who the consumer and provider is. I think a better module would have consumer and provider just be roles that a contract can be associated with, and then you can also have publisher and consumer as the roles for message interactions. I.e. something like Party <-> Role <-> Contract.

3. The contact files and verifications are stored in the database. Pactflow has some customers who republish Pact or OAD files multiple times every day when their CI pipelines run. Some are large (25mb to 50mb) and one customer publishes a 256mb Pact file multiple times a day which never succeeds as it takes longer to process than the 60s AWS ELB and CF allow. The production PactFlow server currently requires 900GB of storage, and the majority of that is in Postgres TOAST tables.

4. The current broker only allows a single file to be published, and it has to be in JSON form (I think for BDCT this was adjusted to allow YAML). There is no way to link different types of files, for instance, with Janus you might publish the contract and the shape document as a separate document linked to it. Some customers try to split their tests up, and then have to combine it all into a single file before publishing where a better process would be to allow them to stage the individual files and then have a final publish request. I played around with a repository prototype inspired by how Git manages thew repositories. It only used the file system and allowed multiple files to be published and linked together with each file individually versioned (i.e. if the same set gets published a second time with only one file changed, only that file gets a new version entry and the index gets updated with all the existing versions for the other files). Prototype is in /development/Projects/Drift/quilt-prototype/quilt-cli/src/repository/

5. The matrix query which drives the can-i-deploy check can easily overload the database.

6. Old versions and results can accumulate and cause excessive DB performance degradation.

7. (Minor) It can only run against Postgres. It used to support MySQL, but that was dropped. But it means you can't run it against a commercial DB.

8. Webhooks can cause a thundering herd problem (i.e. a publish causes a webhook to execute which triggers a number of CI builds which in turn publish something which causes webhooks to fire).

9. Webhooks are push only. Some customers don't want third party systems invoking their CI systems directly.

One of the things that is good with the current Broker is the use of HAL, as it allows the API to evolve easily. When the publishing was changed, a new endpoint was added and all the clients were updated to look for it, and if not there use the old publishing method. This way people using older Pact Broker installations would still work. Also RBAC can be supported by not providing the links for actions the user can not exercise. For instance the UI will not show a delete button if the delete link is not there.

Some things to guide the design:
* It should be modular, so it is easy to add new types of contract files or associated data, and things like user auth and RBAC
* It should be performant.
* It needs to keep concepts like branches because contract tests are mostly run against feature branches in CI.
* There should be consideration for existing Pact Broker users to upgrade to it (i.e. a migration path or data import and a API facade might be a way to address it)
* There should be a configurable lifecycle for anything published from a CI. 

It doesn't have to be implemented in Ruby and written from scratch. We can look at existing libraries and frameworks that might help. It also doesn't need to use a RDMS if another type of storage will work.
