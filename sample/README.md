# StratoWeave YANG-to-TMF mapping

These files are API captures from the SORESPO `quicklab-notconf` test
environment. They show a working implementation of
[draft-lambrechts-onsen-yang-tmf-mapping](../draft-lambrechts-onsen-yang-tmf-mapping-00.xml):
a YANG service schema is exposed as a TMF633 catalog, and YANG service instances
are accepted and returned as TMF640 services.

## Implementation
[SORESPO](https://github.com/stratoweave/sorespo) is a StratoWeave service
orchestrator. Its CFS models mark TMF mapping roots with the extensions from
[`ietf-yang-tmf-map.yang`](../ietf-yang-tmf-map.yang) and independently attach
service transforms with `sw:transform`:

| YANG mapping root | TMF CFS name | Service specification ID |
| --- | --- | --- |
| L3VPN `vpn-service` | L3VPN VPN Service | `/l3vpn-svc/vpn-services/vpn-service` |
| L3VPN `site` | L3VPN Site | `/l3vpn-svc/sites/site` |
| `netinfra/router` | Router | `/netinfra/router` |
| `netinfra/backbone-link` | Backbone Link | `/netinfra/backbone-link` |

StratoWeave reads the YANG schema and CFS data and exposes them through TMF633
and TMF640:

```text
YANG schema + tmf:cfs-service --> TMF633 catalog and specifications
				      |
TMF640 Service JSON <----------> YANG CFS data
				      |
				      v
			 inter/RFS/device layers
```

The implemented representation follows the draft directly:

* An annotated YANG container or list becomes a TMF
	`CustomerFacingServiceSpecification` and a TMF640 `Service`.
* Leaves directly below the mapping root become `specCharacteristic` and
	`serviceCharacteristic` entries.
* Nested containers and lists become `featureSpecification` and `feature`
	entries. Their leaves become feature characteristics, and path-based parent
	relationships preserve the YANG hierarchy.
* YANG list keys are mandatory characteristics and form stable service IDs,
	for example `/netinfra/router[name='AMS-CORE-1']`. Compound keys produce one
	predicate per key leaf.
* YANG types map to TMF `boolean`, `string`, `integer`, or `number` values;
	leaf-lists use the corresponding `*Array` type. Characteristic specifications
	also carry constraints such as cardinality, defaults, patterns, and ranges.

For a TMF640 create, `serviceSpecification.id` selects the YANG mapping root.
StratoWeave validates the characteristics, creates the CFS subtree, derives the
key-qualified service ID, and runs the SORESPO transforms. A TMF640 read maps
the CFS data back to service characteristics. TMF640 is therefore a view of the
same YANG services, not a separate data model.

## Captures

* [`tmf633-service-catalog.json`](tmf633-service-catalog.json) contains the
	single default `ServiceCatalog`, which references the default category.
* [`tmf633-service-category.json`](tmf633-service-category.json) contains the
	default root `ServiceCategory` and references the four candidates discovered
	from the annotated YANG mapping roots.
* [`tmf633-service-candidate.json`](tmf633-service-candidate.json) contains one
	`ServiceCandidate` per mapping root. Each candidate links its category to the
	corresponding service specification.
* [`tmf633-service-specification.json`](tmf633-service-specification.json)
	contains the four generated `CustomerFacingServiceSpecification` objects.
	It is the schema-level result: characteristics encode leaves and constraints,
	while feature specifications encode nested YANG structure. The L3VPN site
	specification is the deepest example, with 112 feature specifications.
* [`tmf640.json`](tmf640.json) is the instance-level service inventory: routers,
	backbone links, one VPN service, and four L3VPN sites. Site instances show
	nested configuration as path-addressed features. This accumulated test capture
	has 58 records but 14 unique service IDs because the configuration was posted
	repeatedly; consumers should treat `id`, not array position, as identity.

## Reproduce

With `test/quicklab-notconf` running in the SORESPO repository:

```sh
make -C test/quicklab-notconf configure-tmf640
make -C test/quicklab-notconf get-tmf633-service-catalog
make -C test/quicklab-notconf get-tmf633-service-category
make -C test/quicklab-notconf get-tmf633-service-candidate
make -C test/quicklab-notconf get-tmf633-service-specification
make -C test/quicklab-notconf get-config-tmf640
```

The Make targets discover the test container's host API port. The TMF633
responses are generated from the loaded schema; the TMF640 response depends on
the services currently present in that test environment.
