this is a joke, dont ever do this

```rust
#![feature(const_deref)]
#![feature(const_fn_trait_bound)]
#![feature(const_mut_refs)]
#![feature(const_pin)]
#![feature(const_precise_live_drops)]
#![feature(const_trait_impl)]
#![feature(inline_const)]

use core::future::Future;
use core::mem::ManuallyDrop;
use core::pin::Pin;
use core::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};
use core::{mem, ptr};

const unsafe fn waker_clone(_: *const ()) -> RawWaker {
    let raw_waker = RawWaker::new(ptr::null(), &RAW_WAKER_VTABLE);

    raw_waker
}

const unsafe fn waker_wake(_: *const ()) {}

const unsafe fn waker_wake_by_ref(_: *const ()) {}

const unsafe fn waker_drop(_: *const ()) {}

const RAW_WAKER_VTABLE: RawWakerVTable =
    RawWakerVTable::new(waker_clone, waker_wake, waker_wake_by_ref, waker_drop);

const fn block_on<F>(future: &mut F, cx: &mut Context) -> F::Output
where
    F: ~const Future,
{
    loop {
        let future: Pin<&mut F> = unsafe { Pin::new_unchecked(future) };

        match future.poll(cx) {
            Poll::Pending => {}
            Poll::Ready(result) => break result,
        }
    }
}

struct Foo;

impl const Future for Foo {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<Self::Output> {
        Poll::Ready("hi!")
    }
}

const RAW_WAKER: RawWaker = RawWaker::new(ptr::null(), &RAW_WAKER_VTABLE);

// SAFETY: `Waker::from_raw` is not const.
const WAKER: Waker = unsafe { mem::transmute(RAW_WAKER) };
const WAKER_REF: &Waker = &WAKER;

fn main() {
    let output = const {
        // SAFETY: `Context::from_waker` is not const.
        let context: Context = unsafe { mem::transmute(WAKER_REF) };
        let mut context: ManuallyDrop<Context> = ManuallyDrop::new(context);

        let mut future = Foo;

        let output = block_on(&mut future, &mut context);

        output
    };

    println!("{:?}", output);
}
```
