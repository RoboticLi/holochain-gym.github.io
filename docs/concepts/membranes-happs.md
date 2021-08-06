# Concepts >> Membranes and hApps ||60

```js script
import "@rocket/launch/inline-notification/inline-notification.js";
import { html } from "lit";
import {
  HolochainPlaygroundContainer,
  EntryContents,
  EntryGraph,
  CallZomeFns,
  ZomeFnsResults,
  ConductorAdmin,
  DhtCells,
} from "@holochain-playground/elements";

customElements.define(
  "holochain-playground-container",
  HolochainPlaygroundContainer
);
customElements.define("entry-graph", EntryGraph);
customElements.define("entry-contents", EntryContents);
customElements.define("zome-fns-results", ZomeFnsResults);
customElements.define("call-zome-fns", CallZomeFns);
customElements.define("conductor-admin", ConductorAdmin);
customElements.define("dht-cells", DhtCells);
```

**TLDR: one special case for validation rules are membrane rules: they dictate who is allowed to enter a network and who isn't. Also, DNAs can be combined into hApps.**

<inline-notification type="tip" title="Useful reads">
<ul>
<li><a href="https://developer.holochain.org/concepts/2_application_architecture/">Core Concepts: Application Architecture</a></li>
<li><a href="https://en.wikipedia.org/wiki/Semipermeable_membrane">Semipermeable membrane</a></li>
</ul>
</inline-notification>

## Membranes

We have already seen that each `DNA` creates its own network, and that it can exclude agents based on their actions inside the network.

Using the same mechanism of validation rules, we can define a special kind of validation rule in our `DNA`: **validation for an agent trying to join the network**.

When the agents install the `DNA`, they can pass a **`MembraneProof` in the process: a piece of data that is going to be used to validate whether this agent is valid in this network or not**.

This allows Holochain to have private and decentralized networks, with a lot of flexibility around the mechanisms by which one agent is allowed into a network. For example a membrane proof could be a verifyable claim from some authority, a ticket for an event, or the signature of one agent that's already inside the network.

Because of all this, **the membrane rules dictate who gets to see which data in Holochain** (except for the case of private entries). If you have access to a DHT, you can scan all the data that is published there, and copy it as you like. This is why it's really important to design appropriate membranes for your use case, thinking about accessncontrol from the beginning.

## hApps

And what about hApps? They are really important, because at the end of the day they are what the Holochain conductor has to install.

When you define a hApp, you are defining a collection of `DNAs` that should be installed when that hApp is installed. Not only can you define new `DNAs` that the conductor didn't know about, but you can also rely on existing `DNAs` being already present and running.

One of the most important mechanisms that we have available when designing hApps is **cloning `DNAs`**. When you clone a `DNA`, you are using the exact same source code for that `DNA`, but overriding some configuration (its `uid` or `properties` field). These configurations change the hash for that `DNA`, so you are effectively creating a new network with the same functionality as the initial one, but with different configuration, and as a result different rules.

In this way a hApp can define complex interactions between agents: you can have one `DNA` for all the functionality of your app, or have different communities.

## Try it!

In this scenario, we have a hApp with two `DNA` slots: a "lobby" `DNA`, and a "privateChat" one. When the agents install the hApp, only the lobby one is installed, and that one doesn't have any membrane so all the agents can join without any issues.

You can see the list of `Cells` that each `Conductor` has installed in the `Conductor Admin` panel. A `Cell` is an instance of a running `DNA`, paired with the public key of the agent running it.

As we said, the "lobby" `DNA` is open, but the "privateChat" one is not: it has a very strict membrane rule that **only allows the agents that have "supersecretcode" in their membrane proofs**.

We are going to try to create 2 different private group chats with this hApp:

1. Select the "Admin API" tab in the "Conductor Admin" panel.
   - You'll see the "Clone DNA" API - this is what we want.
2. Select the "simulated-app" hApp.
3. Select the "privateChat" DNA slot.
4. Add some random text in the `uid` field.
   - This ensures that different private chats have different DNA hashes, so they become separate networks.
   - In a real hApp we would probably do this with properties.
5. Input "supersecretcode" as membrane proof.
6. Click "Execute"!
   - You should see that the "DHT Cells" panel contains now only the agent that has cloned the `DNA`.
   - This is because we are the only ones that have installed this particular `DNA`.
7. Select the "Cells" tab in the "Conductor Admin" panel.
   - You can see the 2 `DNAs` that are running in this node: the "lobby" one, and the "privateChat" one.
8. Select the lobby `DNA`.
9. Select another agent, and confirm that they have only one cell running in their conductor.
10. Clone the "privateChat" `DNA` with exactly the same `uid` as in step 4.
    - You should now see that the newly created network has two agents: we have joined the first agent that cloned the `DNA`!

Repeat the process with other agents and with another `uid` to create multiple networks with multiple private chats. Have fun!

```js story
const simulatedDna0 = {
  properties: {},
  zomes: [
    {
      name: "entries2",
      entry_defs: [
        {
          id: "sample",
          visibility: "Public",
        },
      ],
      validation_functions: {},
      zome_functions: {},
    },
  ],
};
const simulatedDna1 = {
  properties: {},
  zomes: [
    {
      name: "entries",
      entry_defs: [
        {
          id: "sample",
          visibility: "Public",
        },
      ],
      validation_functions: {
        validate_create_agent: (hdk) => async ({ membrane_proof }) => {
          return {
            resolved: true,
            valid: membrane_proof === "supersecretcode",
          };
        },
      },
      zome_functions: {},
    },
  ],
};
const simulatedHapp = {
  name: "simulated-app",
  description: "",
  slots: {
    lobby: {
      dna: simulatedDna0,
      deferred: false,
    },
    privateChat: {
      dna: simulatedDna1,
      deferred: true,
    },
  },
};
export const Simple = () => {
  return html`
    <holochain-playground-container
      .numberOfSimulatedConductors=${10}
      .simulatedHapp=${simulatedHapp}
      @ready=${(e) => {
        const conductor = e.detail.conductors[0];

        const cellId = conductor.getAllCells()[0].cellId;

        e.target.activeAgentPubKey = cellId[1];
      }}
    >
      <div
        style="display: flex; height: 600px; flex-direction: row; margin-bottom: 20px;"
      >
        <conductor-admin style="flex-basis: 600px; margin-right: 20px;">
        </conductor-admin>
        <dht-cells style="flex: 1;"> </dht-cells>
      </div>
    </holochain-playground-container>
  `;
};
```
