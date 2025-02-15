# Selected hooks description

This is a brief documentation for a few selected hooks.
Though hooks & handlers differ in what they are there to do, it is not necessary to describe them all, because the mechanism is general.
The following is meant to give you the idea of how the hooks work, what they are used for and the various purposes they can serve.

## `user_send_packet`

```erlang
mongoose_hooks:user_send_packet(Acc, FromJID, ToJID, El),
```

This hook is run in `ejabberd_c2s` after the user sends a packet.
Some rudimentary verification of the stanza is done once it is received from the socket:

- if present, the `from` attribute of the stanza is checked against the identity of the user whose session the process in question serves;
  if the identity does not match the contents of the attribute, an error is returned,
- the recipient JID (`to` attribute) format is verified.

The hook is not run for stanzas which do not pass these basic validity checks.
Neither are such stanzas further processed by the server.
The hook is not run for stanzas in the `jabber:iq:privacy` or `urn:xmpp:blocking` namespaces.

This hook won't be called for stanzas arriving from a user served by a federated server (i.e. on a server-to-server connection handled by `ejabberd_s2s`) intended for a user served by the relevant ejabberd instance.

It is handled by the following modules:

* [`mod_caps`](../modules/mod_caps.md) - detects and caches capability information sent with certain messages for later use.

* [`mod_carboncopy`](../modules/mod_carboncopy.md) - if the packet being sent is a message, it forwards it to all the user's resources which have carbon copying enabled.

* [`mod_event_pusher`](../modules/mod_event_pusher.md) - if configured, sends selected messages to an external service.

* [`mod_inbox`](../modules/mod_inbox.md) - if configured, stores the message in the user's inbox.

* [`mod_mam`](../modules/mod_mam.md) - stores outgoing messages in an archive.

* [`mod_ping`](../modules/mod_ping.md) - upon reception of every message from the client, this module (re)starts a timer;
 if nothing more is received from the client within 60 seconds, it sends an IQ ping, to which the client should reply - which starts another timer.

* `mod_smart_markers` - checks if the stanza contains chat markers info and stores the update.

## `user_receive_packet`

```erlang
mongoose_hooks:user_receive_packet(StateData#state.host_type, Acc,
                                   StateData#state.jid, From, To, FixedEl),
```

The hook is run just before a packet received by the server is sent to the user.

Prior to sending, the packet is verified against any relevant privacy lists (the mechanism is described in [XEP-0016: Privacy Lists][privacy-lists]).
The privacy list mechanism itself is not mandatory and requires `mod_privacy` to be configured; otherwise all stanzas are allowed to pass.

This hook won't run for stanzas which are destined to users of a different XMPP domain served by a federated server, connection to which is handled by `ejabberd_s2s`.

It is handled by the following modules:

* [`mod_caps`](../modules/mod_caps.md) - detects and caches capability information sent with certain messages for later use.

* [`mod_carboncopy`](../modules/mod_carboncopy.md) - if the received packet is a message, it forwards it to all the user's resources which have carbon copying enabled.

## `filter_packet`

```erlang
mongoose_hooks:filter_packet({OrigFrom, OrigTo, OrigAcc, OrigPacket})
```

This hook is run by `mongoose_router_global` when the packet is being routed by `ejaberd_router:route/3`.
It is in fact the first call made within the routing procedure.
If a function hooked in to `filter_packet` returns `drop`, the packet is not processed.

The `ejaberd_router:route/3` is the most general function used to route stanzas across the entire cluster, and calls to it are scattered all over MongooseIM code.
`ejabberd_c2s` calls it after it receives a packet from `ejabberd_receiver` (i.e. the socket) and multiple modules use it for
sending replies and errors.

The handlers expect no arguments other than the accumulator, which is a packet.
It may or may not be filtered out (in case the handler chain returns `drop`) or modified.

Note the hook code inside `mongoose_hooks`:
```erlang
filter_packet(Acc) ->
    run_global_hook(filter_packet, Acc, []).
```
This hook is run not for a host type, but globally across the whole cluster.
Keep that in mind when registering the handlers and appropriately use the atom `global` instead of a host type as the second argument.

## `offline_message_hook`

```erlang
mongoose_hooks:offline_message_hook(Acc, From, To, Packet);
```

`ejabberd_sm` runs this hook once it determines that a routed stanza is a message and while it ordinarily could be delivered, no resource (i.e. device or desktop client application) of its recipient is available online for delivery.

The following depends on configuration, but the hook is first handled by `mod_offline_chatmarkers`, which filters out and chat marker packets processed by `mod_smart_markers`.
The handler stores such messages and returns `{stop, Acc}`, which terminates the call and no more hook handlers are called.
In case of other packets, `mod_offline` should store the message in a persistent way until the recipient comes online, and the message can be successfully delivered.
Similarly, it stops the other hooks from running after that.

If the `mod_offline` handler fails to store the message, we should notify the user that the message could not be stored.
To this end, there is another handler registered, but with a greater sequence number, so that it is called after `mod_offline`.
If `mod_offline` fails, `ejabberd_sm:bounce_offline_message` is called, and the user gets their notification.

In the case of using `mod_mam` the message is actually stored and no notification should be sent - that's why the module `mod_offline_stub` can be enabled.
It can handle the hook before `ejabberd_sm`, preventing it from sending `<service-unavailable/>`.

## `remove_user`

```erlang
mongoose_hooks:remove_user(LServer, Acc, LUser)
```

`remove_user` is run by `ejabberd_auth` - the authentication module - when a request is made to remove the user from the database of the server.
This one is rather complex, since removing a user requires many cleanup operations:
`mod_last` removes last activity information ([XEP-0012: Last Activity][XEP-0012]);
`mod_mam` removes the user's message archive;
`mod_muc_light` quits multi-user chat rooms;
`mod_offline` deletes the user's offline messages;
`mod_privacy` removes the user's privacy lists;
`mod_private` removes the user's private xml data storage;
`mod_pubsub` unsubscribes from publish/subscribe channels;
and `mod_roster` removes the user's roster from the database.

## `node_cleanup`

```erlang
mongoose_hooks:node_cleanup(Node)
```

`node_cleanup` is run by a `mongooseim_cleaner` process which subscribes to `nodedown` messages.
Currently, the hook is run inside a global transaction (via `global:trans/4`).

The job of this hook is to remove all processes registered in Mnesia.
MongooseIM uses Mnesia to store processes through which messages are then routed - like user sessions or server-to-server communication channels - or various handlers, e.g. IQ request handlers.
Those must obviously be removed when a node goes down, and to do this the modules `ejabberd_local`, `ejabberd_router`, `ejabberd_s2s`, `ejabberd_sm` and `mod_bosh` register their handlers with this hook.

Number of retries for this transaction is set to 1 which means that in some situations the hook may be run on more than one node in the cluster, especially when there is little garbage to clean after the dead node.
Setting retries to 0 is not good decision as it was observed that in some setups it may abort the transaction on all nodes.

## `session_opening_allowed_for_user`
```erlang
allow == mongoose_hooks:session_opening_allowed_for_user(HostType, JID).
```

This hook is run after authenticating when user sends the IQ opening a session.
Handler function are expected to return:

* `allow` if a given JID is allowed to open a new sessions (the default)
* `deny` if the JID is not allowed but other handlers should be run
* `{stop, deny}` if the JID is not allowed but other handlers should **not** be run

In the default implementation the hook is not used, built-in user control methods are supported elsewhere.
This is the perfect place to plug in custom security control.

## Other hooks

* adhoc_local_commands
* adhoc_sm_commands
* amp_check_condition
* amp_determine_strategy
* amp_verify_support
* anonymous_purge_hook
* auth_failed
* c2s_broadcast_recipients
* c2s_filter_packet
* c2s_preprocessing_hook
* c2s_presence_in
* c2s_remote_hook
* c2s_stream_features
* c2s_unauthenticated_iq
* c2s_update_presence
* can_access_identity
* can_access_room
* caps_recognised
* check_bl_c2s
* custom_new_hook
* disco_info
* disco_local_features
* disco_local_identity
* disco_local_items
* disco_muc_features
* disco_sm_features
* disco_sm_identity
* disco_sm_items
* does_user_exist
* ejabberd_ctl_process
* failed_to_store_message
* filter_local_packet
* filter_packet
* filter_room_packet
* find_s2s_bridge
* forbidden_session_hook
* forget_room
* get_key
* get_mam_muc_gdpr_data
* get_mam_pm_gdpr_data
* get_personal_data
* get_room_affiliations
* inbox_unread_count
* invitation_sent
* is_muc_room_owner
* join_room
* leave_room
* mam_archive_id
* mam_archive_message
* mam_archive_size
* mam_get_behaviour
* mam_get_prefs
* mam_lookup_messages
* mam_muc_archive_id
* mam_muc_archive_message
* mam_muc_archive_size
* mam_muc_flush_messages
* mam_muc_get_behaviour
* mam_muc_get_prefs
* mam_muc_lookup_messages
* mam_muc_remove_archive
* mam_muc_set_prefs
* mam_remove_archive
* mam_set_prefs
* mod_global_distrib_known_recipient
* mod_global_distrib_unknown_recipient
* node_cleanup
* offline_groupchat_message_hook
* offline_message_hook
* packet_to_component
* presence_probe_hook
* privacy_check_packet
* privacy_get_user_list
* privacy_iq_get
* privacy_iq_set
* privacy_updated_list
* pubsub_create_node
* pubsub_delete_node
* pubsub_publish_item
* push_notifications
* register_command
* register_subhost
* register_user
* remove_domain
* remove_user
* resend_offline_messages_hook
* rest_user_send_packet
* room_packet
* roster_get
* roster_get_jid_info
* roster_get_subscription_lists
* roster_get_versioning_feature
* roster_groups
* roster_in_subscription
* roster_out_subscription
* roster_process_item
* roster_push
* roster_set
* s2s_allow_host
* s2s_connect_hook
* s2s_receive_packet
* s2s_send_packet
* s2s_stream_features
* session_cleanup
* session_opening_allowed_for_user
* set_presence_hook
* set_vcard
* sm_broadcast
* sm_filter_offline_message
* sm_register_connection_hook
* sm_remove_connection_hook
* unacknowledged_message
* unregister_command
* unregister_subhost
* unset_presence_hook
* update_inbox_for_muc
* user_available_hook
* user_ping_response
* user_ping_timeout
* user_receive_packet
* user_send_packet
* user_sent_keep_alive
* vcard_set
* xmpp_bounce_message
* xmpp_send_element
* xmpp_stanza_dropped

[privacy-lists]: http://xmpp.org/extensions/xep-0016.html
[XEP-0012]: https://xmpp.org/extensions/xep-0012.html
