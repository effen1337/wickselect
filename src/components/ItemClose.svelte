<button class="sv-item-btn" tabindex="-1" data-action="deselect" type="button">
  <svg height="16" width="16" viewBox="0 0 20 20" aria-hidden="true" focusable="false"><path d="M14.348 14.849c-0.469 0.469-1.229 0.469-1.697 0l-2.651-3.030-2.651 3.029c-0.469 0.469-1.229 0.469-1.697 0-0.469-0.469-0.469-1.229 0-1.697l2.758-3.15-2.759-3.152c-0.469-0.469-0.469-1.228 0-1.697s1.228-0.469 1.697 0l2.652 3.031 2.651-3.031c0.469-0.469 1.228-0.469 1.697 0s0.469 1.229 0 1.697l-2.758 3.152 2.758 3.15c0.469 0.469 0.469 1.229 0 1.698z"></path></svg>
</button>

<style>
  .sv-item-btn {
    @apply relative inline-flex items-center self-stretch box-border px-1 border-0 m-0 cursor-pointer bg-error/50 text-error-content;
  }
  .sv-item-btn:hover {
    @apply bg-error/100;
  }
  .sv-item-btn > svg {
    @apply fill-current;
  }
</style>