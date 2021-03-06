<script>
/**
 * Top-level offer / request creation form
 *
 * User flow of submitting listings to a marketplace starts here.
 *
 * :TODO:
 * - Sending proposals to particular agents, rather than the generic broadcast (implement `ProposedTo`)
 * - Proposal.eligibleLocation
 * - Proposal.unitBased and e-commerce-like listing UI
 * - Success message
 * - Error display on page when submission actions fail
 * - Error handling for partial transaction failure (eg Proposal succeeds, Intent fails)
 *
 * @package  ValueFlows UI
 * @since    2020-10-13
 */
import * as yup from 'yup'
import { formup } from 'svelte-formup'
import { mutation } from 'svelte-apollo'

import addPersistence from '@vf-ui/persist-svelte-store'
import BindContextAgent from '@vf-ui/bind-context-agent'

import FieldError from '@vf-ui/form-field-error'

import ListOfferIntent from '@vf-ui/offer-intent-create-form/ListOfferIntent.svelte'
import ListRequestIntent from '@vf-ui/offer-intent-create-form/ListRequestIntent.svelte'

import { createIntent } from '@vf-ui/offer-intent-create-form/queries'
import { createProposal, createProposedIntent, createProposedTo } from './queries'

// An array of vf:Agent identities (usually of Organizations) to use as accounting scopes for records created by this form
export let inScopeOf = null

// set to a string to persist the form state within the given key
export let persistState = false

// (validated) intent objects held after successful child form submission
// index 0 is the primary intent, 1 is the reciprocal intent (if it exists)
let pendingIntents = []

// bindings to child form control submission / validation actions
let intentValidators = []

// API bindings
const runCreateProposal = mutation(createProposal)
const runCreateIntent = mutation(createIntent)
const runCreateProposedIntent = mutation(createProposedIntent)
// const runCreateProposedTo = mutation(createProposedTo) :TODO:

// top-level sub-states of the listing form
const listingTypes = ['gift', 'need', 'offer', 'request']

const formCtx = formup({
  schema: yup.object().shape({
    name: yup.string(),
    note: yup.string(),

    // not VF fields, internal form state only
    listingType: yup.string().oneOf(listingTypes),
  }),
  async onSubmit (data, context) {
    // reset submission state
    pendingIntents = []

    // trigger child form validation
    await Promise.all(intentValidators.map(validator => validator()))

    // post-validation checks
    switch (data.listingType) {
      default:
        // all actions must ensure the presence of a primary intent
        if (pendingIntents.length < 1) {
          return
        }
      // eslint no-fallthrough: 0 */
      case 'offer':
      case 'request':
        // ensure there is at least 1 valid reciprocal intent for bidirectional types
        if (pendingIntents.length < 2) {
          return
        }
        break
    }

    // Validated ok! Run submission behaviours...

    try {
      // create Proposal to broadcast the listing first
      const proposalData = await runCreateProposal({
        variables: { proposal: {
          name: data.name,
          note: data.note,
          hasEnd: new Date(),
          /* eslint no-undefined: 0 */
          inScopeOf: (inScopeOf && inScopeOf.length) ? inScopeOf : undefined,
        } },
      })
      const proposalId = proposalData.data.createProposal.proposal.id

      // create Intents to bind to the Proposal
      const intents = (await Promise.all(pendingIntents.filter(i => i !== undefined).map(i => runCreateIntent({
        variables: { intent: i },
      })))).map(r => r.data.createIntent.intent)

      // create links between all records
      await Promise.all(intents.map((it, idx) => runCreateProposedIntent({
        variables: {
          proposal: proposalId,
          intent: it.id,
          reciprocal: idx === 1, // :SHONK: :TODO: handle arbitrary sets of primary / reciprocal intents, not just a pairing
        },
      })))

      // :TODO: show success indicator in UI
      console.info('SENT!', proposalData.data.createProposal.proposal, intents)

      // clear form state on succcess
      reset()
    } catch (e) {
      // :TODO: nice error display
      console.error(e)
    }
  },
})
const { validate, validity } = formCtx
let { values } = formCtx

// set initial form values
reset()

// inject persistence wrapper to store if configured
if (persistState) {
  values = addPersistence(persistState, values)
}

function updatePrimaryIntent (event) {
  pendingIntents = [event.detail, pendingIntents[1]]
}

function updateReciprocalIntent (event) {
  pendingIntents = [pendingIntents[0], event.detail]
}

function addValidator (ctx) {
  intentValidators = [...intentValidators, ctx.detail.submit]
}
function removeValidator (ctx) {
  intentValidators = intentValidators.filter(cb => cb !== ctx.detail.submit)
}

function reset () {
  $values = { listingType: 'gift' }
}

// form labels (:TODO: move to i18n layer)
const LISTING_TYPE_LABELS = {
  gift: 'Gift something',
  need: 'Ask for something',
  offer: 'Make an offer',
  request: 'Request an exchange',
}

$: console.log('pending intents', pendingIntents)
$: console.log('intent validators', intentValidators)
</script>

<form use:validate>
  <h3>What will you do today?</h3>

  <p use:validity>
    {#each listingTypes as lType}
    <label>
      <input type=radio bind:group={$values.listingType} value={lType} />
      {LISTING_TYPE_LABELS[lType]}
    </label>
    {/each}
    <FieldError at="listingType" />
  </p>

  <p use:validity>
    <label for="name">Name your listing</label>
    <input id="name" bind:value="{$values.name}" type="text" />
    <FieldError at="name" />
  </p>

  <p use:validity>
    <label for="note">Provide a description</label>
    <textarea id="note" bind:value="{$values.note}" placeholder="be as detailed as you like..." />
    <FieldError at="note" />
  </p>

  <hr />

  <BindContextAgent let:contextAgent>
    {#if $values.listingType === 'gift'}
      <ListOfferIntent {contextAgent} persistState={`${persistState}-ointent`}
        on:validated={updatePrimaryIntent} on:initForm={addValidator} on:unloadForm={removeValidator} />
    {:else if $values.listingType === 'need'}
      <ListRequestIntent {contextAgent} persistState={`${persistState}-rintent`}
        on:validated={updatePrimaryIntent} on:initForm={addValidator} on:unloadForm={removeValidator} />
    {:else if $values.listingType === 'offer'}
      <ListOfferIntent {contextAgent} persistState={`${persistState}-ointent`}
        on:validated={updatePrimaryIntent} on:initForm={addValidator} on:unloadForm={removeValidator} />
      <hr />
      <ListRequestIntent {contextAgent} persistState={`${persistState}-rintent`}
        on:validated={updateReciprocalIntent} on:initForm={addValidator} on:unloadForm={removeValidator}
        formTitle="What do you want in return?"
        ACTION_FORM_LABELS={{
          transfer: 'Sell or trade for something else',
          'transfer-custody': 'Borrow something in return',
          work: 'Help doing some work',
          'deliver-service': 'Delivery of a special service',
        }}
        />
    {:else if $values.listingType === 'request'}
      <ListRequestIntent {contextAgent} persistState={`${persistState}-rintent`}
        on:validated={updatePrimaryIntent} on:initForm={addValidator} on:unloadForm={removeValidator} />
      <hr />
      <ListOfferIntent {contextAgent} persistState={`${persistState}-ointent`}
        on:validated={updateReciprocalIntent} on:initForm={addValidator} on:unloadForm={removeValidator}
        formTitle="What are you offering in return?"
        ACTION_FORM_LABELS={{
          transfer: 'Payment or trade',
          'transfer-custody': 'Loan of equipment',
          work: 'Help doing some work',
          'deliver-service': 'Special services',
        }}
        />
    {:else}
      <!-- Invalid listingType: this should never happen! -->
    {/if}
  </BindContextAgent>

  <hr />

  <p>
    <button type="submit">Publish listing</button>
    <button type="reset">Reset</button>
  </p>
</form>

<style>
p, form {
  text-align: left;
}
hr {
  border: 0;
  border-top: 1px solid #DDD;
}
</style>
