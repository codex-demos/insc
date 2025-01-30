1. **Close (or hide) the first modal before—or immediately after—opening the second modal** so there’s only one active modal.
2. **Use `onShown` (or a similar event) on the second modal** to programmatically set focus to one of its buttons.
3. **Verify you have a focus-trap directive (or the modal’s built-in focus-trap) only on the active modal** so focus stays contained in it.

Below are the most common reasons why focus might be leaving the second modal along with some concrete strategies to fix it.

---

## 1. Make sure there is only one _active_ modal

From your code, it looks like you’re opening the success modal (`openSuccessModel(...)`) and _then_ calling `closeModal()`. If both modals are alive for a split second and both have focus-trap logic, you can end up in a race condition where focus ends up on the underlying page or behind the modal.

A straightforward fix is:

```typescript
// Instead of: 
// this.openSuccessModel(this.successTemplate);
// this.closeModal();

// Reverse the order or do them in the callback:
this.closeModal();
this.openSuccessModel(this.successTemplate);
```

Or, if you need to _open_ the success modal first, do something like:

```typescript
this.openSuccessModel(this.successTemplate);

// Give it a quick delay, then close the first modal:
setTimeout(() => this.closeModal(), 0);
```

But in general, best practice is to hide/close the first modal before you show the second one—so you never end up with two “active” modals at once.

---

## 2. Programmatically set focus in the second modal’s `onShown` event

ngx-bootstrap’s `BsModalRef` emits an `onShown` event once the modal is fully rendered. That’s the perfect moment to set initial focus on a button or heading inside the modal.

For example, modify your `openSuccessModel` to do:

```typescript
openSuccessModel(template: TemplateRef<any>) {
  this.successModalRef = this.modalService.show(template, {
    ...this.config,
    ariaDescribedby: 'success-description',
    // If you want to ensure a backdrop or ignore the ESC key, etc.
  });

  // Wait for the modal to finish animating in and attach to the DOM
  this.successModalRef.onShown.subscribe(() => {
    // Focus the close button (or confirm button). Example:
    const closeBtn = document.getElementById('success-close');
    if (closeBtn) {
      closeBtn.focus();
    }
    // Or if you wanted to focus the “Close” or “Yes” button:
    // const yesBtn = document.getElementById('success-confirm-yes');
    // if (yesBtn) {
    //   yesBtn.focus();
    // }
  });
}
```

With this approach, as soon as the success modal is fully open, focus will move to (say) the `success-close` button, and Tab will remain within the modal (assuming focus-trap is active and the first modal is already closed).

---

## 3. Confirm your custom `appFocusTrap` does not conflict

You’ve applied `appFocusTrap` to both modals. Usually that is fine—each modal can have its own focus trap. **But you don’t want two simultaneously “active” traps** (one for the first modal, one for the second). Again, that leads back to closing/hiding the first modal so there is only one active trap.

If you still find that focus is escaping, ensure your `appFocusTrap` directive:

- Is correctly initialized on the second modal’s element.
- Is not forcibly returning focus to the first modal or the background if the first modal is still open.
- Has a way to specify the “initial” focus element or uses `document.activeElement` properly.

---

## 4. Double-check your modal config for `keyboard` or `backdrop` settings

You mentioned using:

```typescript
config: any = {
  backdrop: true,
  keyboard: false,
  ignoreBackdropClick: true,
};
```

- `keyboard: false` prevents closing the modal via `Esc` but it also means that normal built-in focus transitions (like pressing `Tab` or `Shift+Tab`) should still behave as expected _if_ the modal is on top.
- `ignoreBackdropClick: true` just disables closing on backdrop clicks.

That’s all fine. The only optional improvement: if you strictly want to trap focus, you might consider `backdrop: 'static'`, which ensures the user cannot click outside the modal at all. That’s more of a design choice—some want the user to be able to interact with the background.

---

## Putting it all together

1. **Close the first modal before/while opening the second.** Make sure only one is “alive” at once.
    
2. **Use `onShown` on the second modal** to explicitly set focus to, say, the “Close” or “Save” button:
    
    ```typescript
    openSuccessModel(template: TemplateRef<any>) {
      this.successModalRef = this.modalService.show(template, {
        ...this.config,
        ariaDescribedby: 'success-description',
      });
    
      this.successModalRef.onShown.subscribe(() => {
        // Now that the modal is fully shown, set focus
        const successCloseBtn = document.getElementById('success-close');
        successCloseBtn?.focus();
      });
    }
    ```
    
3. **Verify your `appFocusTrap`** doesn’t trap focus on the old/closed modal or require a background click. Usually removing/hiding the first modal is enough.
