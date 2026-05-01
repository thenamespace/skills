# ENSIP-15 Normalization

The single most violated rule in ENS integrations. Read this whenever you accept ENS name input from any source.

**Spec**: [docs.ens.domains/ensip/15](https://docs.ens.domains/ensip/15)
**Canonical library**: [`@adraffy/ens-normalize`](https://github.com/adraffy/ens-normalize) (vendored or wrapped by viem and ethers)

## Why this exists

Unicode is full of attack surface. Naïve approaches like `name.toLowerCase()` are **broken and dangerous**:

- **Homoglyphs** — Cyrillic `а` (U+0430) renders identically to Latin `a` (U+0061). `аpple.eth` and `apple.eth` are different names hashing to different nodes; without normalization rules, both can register and impersonate each other.
- **Zero-width joiners** — invisible `U+200D` characters embedded in a name can change its hash without changing what users see.
- **Confusable scripts** — Greek `ο`, Latin `o`, and Cyrillic `о` look identical.
- **Mapped characters** — uppercase ASCII, fancy quotes, em-dashes, fullwidth digits, Roman numerals (`Ⅵ` vs `vi`) all need to fold to a canonical form.
- **Emoji presentation selectors** — `❤` and `❤️` differ only by the invisible `FE0F` selector.

ENSIP-15 defines exactly one canonical form per name. If two strings normalize to the same output, they are the same name. If a string fails to normalize, it is not a valid ENS name and should be rejected.

## The contract

```
ens_normalize(s: string) -> string  // throws on invalid input
```

- **Idempotent**: `ens_normalize(ens_normalize(x)) === ens_normalize(x)`.
- **Throws** on disallowed characters, malformed emoji sequences, bidi violations, label-boundary violations.
- **Label-aware**: handles the full dotted name, not just one label. Don't split-then-normalize-each-piece — call once on the whole name.

## Where to apply it

At every input boundary:

| Boundary | What to do |
|---|---|
| Search/lookup input from a user | `ens_normalize(input.trim())` before anything else |
| Address-book or saved-contact entry | Normalize at save AND at compare |
| Hashing / namehash computation | Always normalize labels first |
| Comparing two names for equality | Normalize both, then `===` |
| Storing a name in your DB | Store the normalized form; display the beautified form (see below) |
| Receiving a name over an API | Treat it as untrusted input; normalize on receipt |

## Normalize vs beautify (storage vs display)

`@adraffy/ens-normalize` exposes two main functions:

- **`ens_normalize(name)`** — canonical form. Strips emoji presentation selectors (`FE0F`), folds Greek `ξ` to `Ξ` rules per spec. Use for hashing, comparison, storage, registry calls.
- **`ens_beautify(name)`** — restores presentation selectors and applies Greek capitalization. Use for UI rendering.

Storing the normalized form keeps your hashes consistent with onchain state. Displaying the beautified form gives users the colored/full emoji they expect.

```ts
import { ens_normalize, ens_beautify } from '@adraffy/ens-normalize'

const stored = ens_normalize('❤️.eth')   // '❤.eth'  → for hash/lookup
const shown  = ens_beautify(stored)       // '❤️.eth' → for UI
```

Other useful exports:
- **`ens_split(name)`** — array of normalized labels.
- **`ens_tokenize(label)`** — low-level tokenization, useful for debugging "why does this fail?".

## Library wrappers

You usually don't import `@adraffy/ens-normalize` directly — your client library wraps it:

- **viem**: `import { normalize } from 'viem/ens'`
- **ethers v6**: normalization is built into `ensNormalize()` and applied automatically by `provider.resolveName()`.
- **wagmi**: pass already-normalized names to hooks. Use viem's `normalize()`.

Using the library wrapper guarantees you stay in sync if the spec or vendored data updates.

## Right vs wrong patterns

**Wrong**:
```ts
// 1. Lowercase is not normalization
const node = namehash(input.toLowerCase())

// 2. Comparing pre-normalize
if (savedName === userInput) { ... }

// 3. Splitting then normalizing each label
input.split('.').map(label => normalize(label)).join('.')

// 4. Server normalizes, client doesn't (or vice versa)
//    Now the server's hash and the client's hash disagree.

// 5. Catching the throw and falling back to the raw input
try { return normalize(name) } catch { return name }   // accepts invalid names
```

**Right**:
```ts
import { normalize } from 'viem/ens'

// Always normalize the full name in one call. Let it throw on bad input.
const name = normalize(input.trim())

// Compare normalized forms.
if (normalize(savedName) === name) { ... }

// On invalid input, surface a clear error to the user.
try {
  const name = normalize(input)
  // …proceed
} catch (e) {
  showError(`"${input}" is not a valid ENS name.`)
}
```

## Footguns

1. **`toLowerCase()` anywhere on an ENS name.** It's not normalization.
2. **Comparing a user's input to a stored name without normalizing both.** Easy false-negatives.
3. **Treating subnames label-by-label.** Bidi rules and emoji ZWJ sequences are evaluated across the full name; piecewise normalization can silently change semantics.
4. **Catching the normalize-throw.** If normalization rejects a name, it is *not* a valid ENS name. Don't silently accept it.
5. **Server/client mismatch.** If your server rejects names that the client accepts (or vice versa), you have a bypass. Normalize on both sides with the same library.
6. **Confusing "passes normalization" with "visually safe."** A normalized name is *cryptographically* canonical, not visually unambiguous — bidi names and mixed-script labels can still mislead users. Show the truncated address alongside the name in trust-critical UI (sends, signatures).

## Sources

- [ENSIP-15 — Normalization Standard](https://docs.ens.domains/ensip/15)
- [`@adraffy/ens-normalize` on GitHub](https://github.com/adraffy/ens-normalize)
- [`@adraffy/ens-normalize` on npm](https://www.npmjs.com/package/@adraffy/ens-normalize)
- viem: [`normalize()` API](https://viem.sh/docs/ens/utilities/normalize)
