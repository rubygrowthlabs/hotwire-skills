# Reference Index

All reference articles for the `forms-validation` skill. Each article is self-contained with an overview, implementation guidance, and pattern cards showing good vs. bad approaches.

## Hotwire Focus

- Turbo Frames
- Turbo Form Submissions
- Stimulus (for form behavior)

## Articles

| # | File | Topic | Summary |
|---|------|-------|---------|
| 1 | [turbo-frames-inline-edit.md](./turbo-frames-inline-edit.md) | Inline Editing | Click-to-edit pattern using Turbo Frame swaps between display and edit views. Auto-focus, submit-on-blur, and state persistence. |
| 2 | [turbo-frames-modal-validation.md](./turbo-frames-modal-validation.md) | Modal Forms | Modal dialogs with form validation using Turbo Frames. Native `<dialog>` element with Stimulus, 422/303 response handling, closing on success. |
| 3 | [turbo-frames-typeahead.md](./turbo-frames-typeahead.md) | Typeahead Search | As-you-type search updating results in a Turbo Frame. Debouncing with Stimulus, loading states, and URL parameter preservation for bookmarkable search. |
| 4 | [turbo-frames-external-forms.md](./turbo-frames-external-forms.md) | External Form Controls | Form elements outside the frame that submit to the frame. The `form` attribute, `data-turbo-frame` targeting, and split-layout form patterns. |
| 5 | [form-submission-lifecycle.md](./form-submission-lifecycle.md) | Submission Lifecycle | Complete Turbo form submission lifecycle from submit to response rendering. 422 vs 303 status codes, turbo:submit-start/end events, and button state management. |
| 6 | [action-parameters-forms.md](./action-parameters-forms.md) | Stimulus Action Parameters | Passing data from form elements to Stimulus actions via params. Dynamic form behavior, conditional fields, and element-specific actions without manual data attribute parsing. |
