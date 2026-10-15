# Double-click gesture support

## Scope and baseline

Initial delivery supports the C++ API with Qt Quick on Windows, Qt 6.11.0,
and static builds. Qt Widgets double-click and RPC registration are follow-up
work. Existing Qt Widgets operations must continue to build and pass tests.

Base: v0.14, `7ab2831b0ce78b779805bd54ee97c880738cf8f6`.

## Implementation

1. Build the unchanged baseline with `SPIX_BUILD_TESTS=ON`.
2. Add a separate GUI test executable with a generic nested selection target.
   Use `QTest::mouseDClick(QWindow*)` as the positive control and ordinary
   Spix clicks as the negative control.
3. Implement one complete Qt Quick gesture: press, release, press,
   double-click, release. Post events in order without sleeps, timers, or
   processing the application event loop inside the production backend.
4. Add `DoubleClickOnItem`, reusing `ItemPosition` and executor error reporting.
5. Add four `TestServer::mouseDoubleClick` overloads matching `mouseClick`:
   path; path/proportion; path/proportion/offset; path/button/modifiers.
6. Add a non-pure `Events::mouseDoubleClick` method returning whether the
   gesture was queued. The default returns false, and the command reports
   the failure. This permits existing backend subclasses to compile without
   silently pretending to support the gesture. Rebuild binary consumers.

Default: left button, no modifiers, item center. Supported buttons are one
of left, right, or middle. Existing modifier flags may be combined. Invalid
input, a missing target/window, unsupported backend, or an already-held mouse
button must not start a partial gesture. Success means events were queued;
tests must wait for an observable application result.

## Acceptance

- First click updates selection before exactly one double-click activation.
- Completion releases the button; subsequent click and drag still work.
- Center, proportion, offset, nested/transformed coordinates, all advertised
  buttons, and modifiers are covered.
- Missing targets, unsupported backends, invalid arguments, and target
  destruction do not crash or corrupt subsequent mouse input.
- A separate GUI test process uses `QGuiApplication`; existing non-GUI tests
  retain their application setup.
- Run the gesture suite 100 consecutive times, stopping on failure. Temporarily
  disable the gesture implementation to demonstrate regression sensitivity.
- Existing tests and formatting checks pass. Record actual results below;
  compilation is not behavioral coverage.

## Validation record

Implemented: C++ command/API, mock backend, Qt Quick gesture, command tests,
and a separate GUI test executable. The release tag identifies the immutable
revision accepted by the release gates.

Build configuration: MSVC 14.50.35717, Qt 6.11.0, Windows SDK 10.0.26100.0,
Ninja, Release, `BUILD_SHARED_LIBS=OFF`, `SPIX_BUILD_TESTS=ON`,
`SPIX_BUILD_EXAMPLES=OFF`, both scene libraries enabled.

Results:

- Unchanged v0.14: all three existing CTest suites pass.
- Feature: all four CTest suites pass, including eight new command tests and
  nine real GUI cases. GUI coverage includes a TapHandler as well as MouseArea.
- All nine GUI cases pass 100 consecutive CTest runs (stop-on-failure).
- Repeated TapHandler gestures receive increasing event timestamps, separated
  past the double-click interval as in Qt's test helper. This separates explicit
  gestures without delaying delivery; constant zero timestamps would accumulate
  later taps into the first gesture and suppress subsequent doubleTapped signals.
- An isolated build with the backend disabled compiles successfully, passes
  the Qt positive control, and fails the Spix behavioral assertion with zero
  double-click activations. The working checkout is not changed by this check.
- Release events use `Qt::NoButton` for the held-button mask. The GUI test
  compares this with Qt's positive control; clearing the mask also releases
  pointer-handler grabs. Existing single-click implementation is unchanged.

## C++ usage

From a test server's worker thread:

```cpp
mouseDoubleClick("main/row");
mouseDoubleClick("main/table", {0.5, 0.25});
mouseDoubleClick("main/table", {0.0, 0.0}, {20.0, 30.0});
mouseDoubleClick("main/row", spix::MouseButtons::Right, spix::KeyModifiers::Control);
```

These calls enqueue work; wait for the application's resulting state with a
bounded timeout before asserting completion. Inspect the usual executor errors
for missing items, invalid arguments, or unsupported backends. Earlier ordinary
click callbacks are part of a double-click and are not themselves failures.

## Follow-up

- Implement and behaviorally test Qt Widgets gesture delivery.
- Register explicit RPC methods and test argument types/counts and help text.
- Validate the current upstream Qt 5/Qt 6, OS, static/shared, and install matrix.
- Document source/binary compatibility and asynchronous execution semantics.
- Prepare a focused upstream PR referencing issue #116 with reproducible
  generic examples and precise tested/untested configuration claims.

Keep release commits stable. The initial neutral release tag is
`double-click-qtquick-1`, created only after release validation succeeds.
