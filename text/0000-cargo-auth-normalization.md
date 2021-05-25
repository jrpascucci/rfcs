- Feature Name: cargo_auth_normalization
- Start Date: 2021-05-19
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#9494](https://github.com/rust-lang/rust/issues/9494)

# Summary
[summary]: #summary

Implement https://github.com/arlosi/rfcs/blob/always-auth/text/0000-always-auth.md but also allow overrides and additional functionality.

Following the credentials-process, enable private registries by normalizing and expanding the handling of authentication across registries and downloads in a future-proof way.

# Motivation
[motivation]: #motivation

Initially, the issue was a jfrog private artifactory behind a proxy almost, but didn't quite, work. 

Delving into it, some inconsistencies were noticed, several other multi-year open RFCs have been written, and it's time to stick a fork in it.

This makes crates.io a more equal-among-peers, and allows different repos to have radically different access requirements.

JFrog just released its stuff a month ago or so, knowing of this outstanding issue: they took a good gamble on implementing crates support, 
we can do our side, and can afford to repay dividends without disproportion or negatively affecting future growth or change.

The particular benefit to these changes are floating on the back of the recently approved and implemented RFC [rust-lang/rfcs#194] 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Enterprises with tight security policies around who can read and write source who endeavor to maintain 
logged and authenticated access to their source use are a bit screwed.

This proposal adds the ability to configure different headers associated with each registry, and to 
populate header values with credentials.toml or credentials-process generated tokens under https.

Example: 

~/.cargo/config.toml

```[registry]
default = "my-special-registry"

[registries]
my-special-registry = { index = "https://server/somepath.git" } # contains a mirror of crates.io 
that has been deemed 'blessed', plus some enterprise-common source.
crates-io-direct = { index = "http://crates.io" } # second level availability for development, but can't be used for production.
my-super-secret-registry = { index = "" } # the secret sauce, so secret, it doesn't have a config.json
my-secret-git = { index = "ssh://server/someotherpath.git" } # the other secret sauce, protected by ssh

[source.crates-io]
replace-with = "my-special-registry"

[registries.crates-io-direct]
 # In production formal build, use a bogus name
 # replace-with = "do_not_check_in_crates_io_dependencies_jerkface"
registry = { index = "http://crates.io" } 
http = {proxy = "socks5://myexternalproxy:8908"}

[source.my-special-registry]
registry = "https://server/somepath.git"
http = { proxy = "socks5://myinternalproxy:8080" }

[source.my-super-secret-registry]
registry = "https://sserver/somepath.git"
dl = "https://sserver/artifactory/api/cargo/crates-io/v1/crates"
api = "https://sserver/artifactory/api/cargo/crates-io"
http = { proxy = "socks5://myinternalproxy:8080" }


# Even if the super-secret-registry returned some values for dl and api, we're overriding them.
[source.my-secret-git]
dl = "https://server/someotherotherpath.git"
api = "https://server/someotherotherpath.git"
net = {git-fetch-with-cli = true}
```
Credentials.toml:
```
[registries.my-super-secret-registry]
# not gorgeous, but coherent.
credential-process = "cargo-credential-gnome-secret get | awk '{print \"X-Auth-Token: \\\"\"$0\"\\\"\"; print \"SpecialHeader: \\"foobar\\"}'"

[registries.my-special-registry.update]
headers = ["Authentication"= "Bearer biglongtoken"]

[registries.my-special-registry.index]
headers = ["Authentication": "simpletoken"]

[registries.my-special-registry.download]
headers = ["Authentication": "simpletoken"]
```


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Too much work.

We like things when they are specialized and inconsistent.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Future-proof to some significant extent. Removes additional hard-coded behaviors.
- Relatively limited change
- In the space of all possible designs, lots are way too much work.
- A bigger security revamp was considered, but it seemed overkill.
- If we don't do it, chaos reigns and fire will consume the earth.

# Prior art
[prior-art]: #prior-art

It's basically what we're doing for indexes, extended to downloads.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

I don't immediately think there are any conflicts with the existing config , although transition towards this mechanism as preferred should be managed, perhaps over Rust 2021.

If there's any case where we really wouldn't want to do the credentials initialization on start, or whether we should use futures.

The bigger security concerns are out of scope - public/private key credentials, certificate validation can be added in a future RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

It's very possible we won't have to mess with this for the forseeable future.

But, likely, deeper, pluginable crypto integration is a thing that might be needed some day, specifically hooked up to instances and containers with unique crypto mechanisms that are running in cloud farms (e.g. AWS' IAM security model, SSO and multifactor security models)
