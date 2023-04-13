# Party on Two Participant Nodes

Consult the documentation for deeper understanding: https://docs.daml.com/canton/usermanual/identity_management.html#party-on-two-nodes.

Contributors:
 - [tamaskalcza-da](https://github.com/tamaskalcza-da)
 - [mjuchli-da](https://github.com/mjuchli-da)

## Prerequisites
- Docker with access to Digital Asset enterprise repository
- Maven
- Daml SDK

1. Build the project

   ```shell
   mvn compile
   ```

2. Start `participantA`

   ```shell
   docker compose up -d participantA
   ```

   Optionally you can override the environment variables with for example a `.env.local` file and use it like:

   ```shell
   docker compose --env-file .env.local up -d participantA
   ```

3. Setup `participantA`
    1. Allocate party `Alice`
       ```
       daml ledger allocate-parties Alice --host localhost --port 5011
       ```
    3. Upload the [models DAR]
       ```
       daml ledger upload-dar --host localhost --port 5011 target/multi-node-canton-example-0.0.1.dar
       ```
    5. Create one or more [`MultiNode.Account`] contracts, e.g. using the navigator
4. Start `participantB`

   ```shell
   docker compose up -d participantB
   ```

   Optionally you can override the environment variables with for example a `.env.local` file and use it like:

   ```shell
   docker compose --env-file .env.local up -d participantB
   ```

5. Setup `partipantB`
    1. Upload the [models DAR]
6. Do the party migration

   You need to run the following set of commands using respective interactive Canton consoles. You access the consoles by attaching to the Docker instances.

   ```shell
   docker attach participantA
   ```

   ```shell
   docker attach participantB
   ```

   If you want to detach from a console use `Ctrl+P Ctrl+Q` combination as described in the [Docker manual](https://docs.docker.com/engine/reference/commandline/attach/#description).

   Commands on `participantB`:

   ```scala
   val alice = multiNodeDemo.parties.list().find(x => x.party.toString.contains("Alice")).get.party
   participantB.topology.party_to_participant_mappings.authorize(TopologyChangeOp.Add, alice, participantB.id, RequestSide.To, ParticipantPermission.Submission)
   participantA.topology.party_to_participant_mappings.authorize(TopologyChangeOp.Add, alice, participantB.id, RequestSide.From, ParticipantPermission.Submission)
   ```

   Commands on `particpantA`:

   ```scala
   // Using string values is not ideal, need a better solution.
   val alice = multiNodeDemo.parties.list().find(x => x.party.toString.contains("Alice")).get.party
   val timestamp = participantA.topology.party_to_participant_mappings.list(filterStore = "multiNodeDemo", filterParty = "Alice").map(_.context.validFrom).max
   participantA.repair.download(Set(alice), "/migration/alice.acs.gz", filterDomainId = "multiNodeDemo", timestamp = Some(timestamp))
   ```

   Commands on `participantB`:

   ```scala
   // This step is needed, otherwise `participantB` will not accept the ACS import.
   participantB.domains.connect(multiNodeDemo.defaultDomainConnection)
   // This step is needed, otherwise an error is thrown stating that `participantB` is still connected.
   participantB.domains.disconnect_all()
   repair.party_migration.step2_import_acs(participantB, "/migration/alice.acs.gz")
   participantB.domains.reconnect_all()
   ```

7. Try `participantB`

   You can confirm (e.g. via REPL) that contracts are visible.

   ```shell
   daml repl --ledger-host localhost --ledger-port 5021 target/multi-node-canton-example-0.0.1.dar
   ```

   In the Daml REPL run:

   ```shell
   import MultiNode.Account
   listKnownParties
   let alice = partyFromText("Copy the party ID here.")
   accounts <- query @Account alice
   show accounts
   ```

[models DAR]: target/multi-node-canton-example-0.0.1.dar
[`MultiNode.Account`]: src/main/daml/MultiNode/Account.daml
