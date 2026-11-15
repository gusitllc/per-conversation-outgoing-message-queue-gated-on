# Per-Conversation Outgoing Message Queue Gated on Clean Completion of the Preceding Streamed Answer

**Defensive Publication** - published to establish prior art.
**Publisher:** Gus IT LLC.

## Abstract

A chat client lets the user keep writing while a streamed model answer is in progress. Messages written during streaming join a per-conversation queue instead of being sent or blocked. The next queued message is sent automatically only if the preceding turn ended cleanly: the screen is idle, the last message is an assistant answer, and that answer was not stopped, failed, refused, closed or still pending. Any other ending pauses the queue behind an explicit user action that shows the number waiting. A user stop marks the current answer and registers the stop's own promise, and nothing queued is sent until it settles. A refused or rate-limited message returns to the head of the queue while the user's current draft is preserved. A new conversation's provisional queue is merged into its real key once the server assigns an identifier, and nothing is ever sent offline.

## Technical Field

Client-side message handling for conversational interfaces that stream responses from a language model.

## Background

In chat interfaces that stream model answers, users often think of the next question before the current answer has finished. Clients commonly handle this in one of two ways: they disable the input until the answer ends, which loses the user's train of thought, or they send the new message at once, which interleaves turns, competes with the running answer and may be rejected by the server. A simple first-in-first-out queue that drains after every turn improves on both but sends queued text even when the preceding answer was stopped by the user, refused, cut off by a network error or rate-limited, so a follow-up question is attached to an answer the user never accepted. Races also arise around the stop action, whose server acknowledgement arrives later than the local state change, and around new conversations, whose identifier is assigned by the server only after the first turn, so a queue keyed on the identifier can be orphaned.

## Summary of the Disclosure

The disclosed client keeps an in-memory map of queues, one per conversation, each holding an ordered list of items (text and finished attachments) and a paused flag. While an answer streams, the send control adds the composer content to the queue and empties the composer. After every turn a single function decides what happens next: it sends the head item only if the previous answer ended whole and the device is online; otherwise it pauses the queue and shows a control labelled with the count of waiting items. A stop issued during streaming flags the reply and records the stop's promise, which must settle before any queued item is sent. Items sent and then refused or rate-limited return to the head while the draft the user was typing is kept. Leaving the conversation with items waiting asks the user to send, keep or discard them.

## Detailed Description

Queue identity. Each conversation is identified by a kind and an identifier. For an existing conversation the queue key is that pair. For a new conversation that has no server identifier yet, the screen mints a provisional suffix from a counter, so two new conversations never share a queue. Each time the queue is looked up, the current key is computed and compared with the key last used by the screen; when they differ, because the server has now assigned an identifier, the items under the old key are appended after any items already under the new key, the paused flags are combined, and the old key is removed. Queues are held in memory only and are cleared at sign-out.

Enqueueing. While an answer is streaming, the composer shows a queue button beside the stop button; it is disabled when offline or when the composer holds neither text nor a finished attachment. Sending while streaming, while a stop is being awaited, or while unpaused items are still waiting, appends an item with a unique identifier, the text and the finished attachments to the end of the queue, empties the composer and the saved draft for the conversation, and announces the new count to assistive technology. If the screen is idle and no stop is pending, the oldest item is sent immediately.

Clean-completion test. A turn counts as whole only if the screen phase is idle, no stop is in progress or awaited, the conversation was not refused or deleted, the last message exists and is from the assistant, and that message carries none of the markers for error, stop (including a stop during queueing), closed or pending. After every turn the client checks: if the queue is empty or paused nothing happens; if the screen has gone away, the queue is paused so the next visit shows the control; if the turn was not whole or the device is offline, the queue is paused and a status line states how many items wait; otherwise the next item is sent.

Stop synchronisation. When the user stops a streaming answer, the client marks the last assistant reply with a queue-stop marker, so that its end can never be judged whole, and calls the stop action. If the action returns a promise, the client stores a hold derived from it that resolves whether the stop succeeds or fails, and clears the hold when it settles if it is still the current one. Whenever the client is about to send a queued item and a hold exists, it instead sets a waiting flag, announces that the queue will continue after the stop finishes, and waits on the hold. When the hold settles, the client sends only if items remain, the queue is not paused, no answer is streaming, the screen is still present and the device is online; otherwise it pauses.

Sending one item and handing back. To send, the client removes the head item, clears the paused flag, saves whatever the user is currently typing as the conversation's draft, and calls the screen's turn function with the item. When that turn settles, if the conversation recorded a refusal or entered the rate-limited phase (which the send layer sets on a too-many-requests response, together with a retry-after time), the item is put back at the head of the queue and the composer is restored to the saved draft rather than to the queued text. The after-turn function then runs again. An optional confirmation gate may be supplied by the screen; when it declines to run the send or leaves a question open, the queue pauses with the item still at its head.

Resuming and editing. The explicit resume control refuses while offline, and while a server-requested wait time has not elapsed, with a status message and nothing sent. While an answer streams it only clears the paused flag, so the item goes when that answer ends whole. Each queued row can be removed, or taken back into the composer for editing, in which case it is placed above any text already typed so nothing is lost.

Leaving the conversation. A route guard runs before navigation. If the conversation no longer exists or access was denied, its queue is dropped. Moving to a voice or microphone screen of the same conversation is allowed without a question. Otherwise, with items waiting, navigation is held and a dialog offers to send them, which stays and resumes; to keep them, which pauses the queue and leaves; or to discard them, which deletes the queue and leaves.

## Disclosed Features (each disclosed alone and in combination)

1. A method in which messages composed while a model answer is streaming are appended to a per-conversation in-memory queue rather than being sent or blocked.
2. A method in which a new conversation's queue is held under a provisional per-screen key and merged into the conversation's server-assigned key, preserving order, when that key becomes known.
3. The method in which, after every turn, the next queued message is sent only if the screen is idle, the last message is an assistant answer, and that answer carries no stop, error, refusal, closed or pending marker.
4. The method in which any other turn ending pauses the queue behind an explicit resume control that shows the number of waiting messages.
5. A method in which a user stop during streaming marks the current reply so that its ending is not treated as clean, and records the stop's promise so that no queued message is sent until that promise settles.
6. The method in which a queued message that is refused or rate-limited is returned to the head of the queue while the composer keeps the draft the user was typing.
7. The method in which a manual resume is refused while offline or while a server-requested wait time has not elapsed, and queued messages are never sent while offline.
8. A system in which leaving a conversation with queued messages prompts the user to send, keep or discard them, except when moving to another view of the same conversation or when the conversation no longer exists.
9. The system in which an optional per-send confirmation step may decline a queued send, leaving the message at the head of a paused queue.
10. The system in which a queued message can be removed or taken back into the composer above any text already typed.

## Variations

- The queue may be persisted locally so that it survives a restart, with the same clean-completion rule applied on reload.
- The clean-completion test may be computed by the server and delivered as an explicit end-of-turn status instead of being inferred from message markers.
- Queued messages may be merged into a single follow-up turn instead of being sent one at a time.
- The stop hold may be implemented with a timeout so that a stop acknowledgement that never arrives eventually pauses the queue.
- The confirmation step before each queued send may be a policy check, a cost check or a user prompt.

## Note on omitted parameters

Retry-after handling values and any limits on queue length are omitted; they are implementation choices.
