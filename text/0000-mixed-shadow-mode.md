---
title: Mixed Shadow Mode
status: DRAFTED
created_at: 2021-05-07
updated_at: YYYY-MM-DD
pr: https://github.com/salesforce/lwc-rfcs/pull/49
---

# Mixed Shadow Mode

## Introduction

LWC offers a collection of polyfills that implement Shadow DOM features. These polyfills are
collectively referred to as "synthetic Shadow DOM" and are published to npm under the
`@lwc/synthetic-shadow` package. Synthetic shadow DOM provides a consistent experience for LWC web
components across the Salesforce-supported browser matrix.

Native Shadow DOM support has greatly increased since the start of the LWC project but applications
continue to rely on the synthetic shadow polyfills to maintain backwards-compatibility. The removal
of these polyfills can result in broken experiences due to dependencies on certain "escape hatches"
that are not possible in a native Shadow DOM environment (e.g., instrumentation, styling, etc, which
are discussed below).

Applications that wish to use LWC components in a native shadow context simply need to omit the
inclusion of the `@lwc/synthetic-shadow` polyfill; however, this is not a realistic option for
large applications that cannot afford a rewrite.

This proposal opens up an incremental migration path for applications moving to native Shadow DOM by
allowing the usage of both native and synthetic Shadow DOM in the same document.

## Detailed design

Components will opt-in to this new feature and the default mode will be synthetic mode. From the
component's perspective, there are two choices of shadow semantics: synthetic mode or native
mode. The concept of a mixed mode should be transparent and the framework should guarantee this
through integration testing, which is further discussed below.

A component can signal that it prefers native Shadow DOM by setting a static property called
`preferNativeShadow` to `true`.

```js
export default class extends LightningElement {
    static preferNativeShadow = true;
}
```

If this property is not defined, a default value of `false` will be used. The value of this property
will be read once during component construction and cached for the lifetime of the component. The
LWC engine will throw an exception if the value of `preferNativeShadow` is not a boolean value.

All polyfills applied by `@lwc/synthetic-shadow` will look to this property as a signal to invoke
either the polyfilled API or the native browser API. If the browser does not support Shadow DOM
(e.g., IE11), then LWC will fallback to synthetic mode regardless of which mode the component
prefers.

Due to observable differences between the two modes, it is likely that components that prefer native
will need to know whether it is currently operating in native or synthetic Shadow DOM. Any hints
provided by the framework will only be available to components that set the `preferNativeShadow`
property to `true` because this information is not useful to components that only support synthetic
Shadow DOM.

In terms of composition, a synthetic mode component can contain a native mode component, but the
inverse is not allowed. Not only does this make things easier to reason about, but many existing
components rely on workarounds in synthetic mode that are not possible to support in native mode.
These observable differences are further discussed below. As this invariant cannot be asserted
during compile-time, the engine will assert it during runtime.

### Testing

LWC currently has integration tests for synthetic shadow mode and native shadow mode. A third
testing mode called "mixed shadow mode" will be introduced. Synthetic shadow DOM polyfills are
implemented as patches on global prototypes so the decision of invoking a polyfilled API vs a native
API will depend on the preference of individual components during runtime. This test environment
will ensure that components designed to run in a native Shadow DOM environment will work correctly
when even when synthetic Shadow DOM polyfills are present.

Components that prefer native mode will need to also support synthetic mode in browser environments
where Shadow DOM is natively unavailable, so they will also need to be tested in both modes. LWC
test utilities will need to be updated to facilitate this.

The existing WPT (Web Platform Tests) test suite for Shadow DOM APIs will also be run to identify
any coverage gaps in LWC integration tests.

## Observable differences

### Light DOM and assigned elements

In native Shadow DOM, elements are rendered in a component's light DOM and assigned to slots in a
child component's shadow DOM. This means that elements will exist in the DOM regardless of whether
they are assigned to a slot. This is not the case for LWC--elements that are passed down to child
components are never rendered unless they are assigned to a slot.

In the following example, `span` will not exist in the DOM in synthetic shadow, but it will exist in
native shadow.

```html
<!-- x-parent -->
<template>
    <x-child>
        <span>foo</span>
    </x-child>
</template>

<!-- x-child -->
<template>
    <p>child</p>
</template>
```

This observable difference will not be remediated as such a change would be non-trivial and
backwards-incompatible with the current synthetic shadow implementation.

### Lifecycle timing

Due to the way that slotting is implemented in `@lwc/synthetic-shadow`, there is an observable
difference in the timing of lifecycle hooks for slotted elements.

As mentioned previously, in synthetic mode, slotted elements that are never assigned to a slot are
not rendered. This means that their lifecycle hooks are never invoked.

Another difference is that for synthetic mode, lifecycle hooks are invoked in the order of
appearance after they are assigned, whereas in native mode, they are invoked in the order of
appearance in the template.

### this.shadowRoot vs this.template

`this.shadowRoot` and `this.template` will both reference the component's shadow root in native
Shadow DOM while `this.template` currently references the component's shadow root in synthetic
Shadow DOM.  When a component specifies a preference for native Shadow DOM, `this.shadowRoot` will
be made available as an alias to `this.template`.

There are no plans to enable `this.shadowRoot` for existing components that only support synthetic
Shadow DOM as the current LWC-ism is to use `this.template`.

### this.shadowRoot instanceof ShadowRoot

Components that favor native Shadow DOM will observe that `this.shadowRoot instanceof ShadowRoot`
evaluates to `false` in mixed mode. This is due to the fact that the synthetic shadow polyfill
patches and replaces `ShadowRoot`. This will be remediated by customizing the `instanceof` behavior
using `Symbol.hasInstance`.

### Listening for non-composed events above the root LWC node

There currently exists logic that allows listeners to handle non-composed events outside of the root
LWC node if the event originates from a non-LWC component in the subtree. This behavior exists for
legacy reasons, is only possible to allow in a synthetic Shadow DOM environment, and cannot be
preserved in a native Shadow DOM context.

### Accessibility

The most common accessibility issue in LWC is related to id-referencing across shadow boundaries.
The current workaround for this is to bypass the LWC engine's id-mangling by setting attributes
dynamically, but such a workaround would not work in native Shadow DOM.

### Instrumentation

Applications that rely on instrumentation libraries that don't yet support Shadow DOM are currently
able to obtain references to descriptors like `addEventListener()` before they are patched and use
them to override `@lwc/synthetic-shadow` polyfills where needed. Such workarounds would not work for
events originating from components that prefer native Shadow DOM.

### Styling

Applications currently rely on global stylesheets to apply deeply throughout the DOM. With native
Shadow DOM, this would not be possible. All of a component's styles would have to be generated and
added to the component bundle.

## Drawbacks

A potential (non-verified) drawback is that existing components may experience a slight performance
degradation due to the fact that mixed mode requires logic-forking whenever a Shadow DOM API is
invoked.

## Alternatives

Other alternatives have not been considered but contributions to this proposal are welcome.

## Adoption strategy

Component authors must opt-in when implementing components that prefer native shadow in a mixed
shadow DOM environment due to observable differences between synthetic mode and native mode. As
such, it is the responsibility of the component auther to identify and resolve any breaking changes
for existing components.

Note that the current proposal for mixed mode considers polyfills as all-or-nothing. LWC is not in
the business of providing workarounds for browser implementation issues, unless it affects the
functionality of the framework itself.

# How we teach this

Developers should know about mixed mode and the ability for a component to favor native Shadow DOM
before they start implementing new components. This would eliminate any potential tech debt from
accumulating before crossing the start line. Observable differences should be documented to assist
in deciding whether a component is a candidate for native Shadow DOM.

# Unresolved questions

N/A

# Resolved questions

1. Light DOM components - What does the runtime assertion for preventing synthetic mode components
   in the native mode component subtree look like with Light DOM components in play?

   Enforcing the invariant that a synthetic mode component cannot have any native mode component
   ancestors would allow native mode components to contain light DOM components.