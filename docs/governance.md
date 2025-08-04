## Governance Manual

This manual describes the processes to submit a gov proposal and vote a proposal.

As of the writing of this manual, the gov params are set as:

```json
{
  "voting_params":{
    "voting_period":"300000000000"
  },
  "tally_params":{
    "quorum":"0.667000000000000000",
    "threshold":"0.667000000000000000",
    "veto":"0.334000000000000000"
  },
  "deposit_params":{
    "min_deposit":"0",
    "max_deposit_period":"120000000000"
  }
}
```

Note that to update these params themselves, a param change governance process is needed.

We usually propose and vote on two types of governance proposals, 1. simple param changes for a certain module and 2. cbridge / pegbridge config updates. There are
other types of governance proposals such as farming adjustments and software upgrades, but they happen less often.

### Param Change

1. Query current staking syncer duration and submit change proposal:

NOTE: Replace the placeholder {proposal_id} with the real proposal ID in the following commands. You can find the proposal ID in the output after submitting a proposal.

```sh
bvnd query staking params --home ~/.bvnd
bvnd tx gov submit-proposal param-change ./gov-example/param_change_proposal.json --home ~/.bvnd
bvnd query gov proposal {proposal_id} --home ~/.bvnd
```

2. Perform the command below to vote yes and then wait for enough voters to vote yes in `voting_period`:

```sh
echo | bvnd tx gov vote {proposal_id} yes --home ~/.bvnd
```

3. Query proposal status:

```sh
bvnd query gov proposal {proposal_id} --home ~/.bvnd
bvnd query staking params --home ~/.bvnd
```

Edit the subspace and key accordingly in `./gov-example/param_change_proposal.json` for updating the other params in other modules.

### SEE ALSO
