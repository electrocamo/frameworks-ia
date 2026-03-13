---
name: forms-and-validation
description: >
  Read when the task involves forms, input validation, React Hook Form,
  Zod schemas for forms, form error handling, or form UX patterns.
triggers:
  - form
  - validation
  - React Hook Form
  - Zod
  - input
  - field
  - submit
  - register
  - useForm
---

# SKILL: Forms and Validation

## Mandatory stack

- **React Hook Form** for form state management
- **Zod** for the validation schema
- **@hookform/resolvers/zod** to connect both

Never use `useState` to manage form field values.
Never write manual validation without a schema.

---

## Full pattern — form with Zod + RHF

```typescript
// src/features/users/components/CreateUserForm.tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'
import { useCreateUser } from '../hooks/useUsers'

// Zod schema — single source of truth for validation and types
const CreateUserFormSchema = z
  .object({
    name: z
      .string()
      .min(1, 'Name is required')
      .max(100, 'Maximum 100 characters'),
    email: z
      .string()
      .min(1, 'Email is required')
      .email('Invalid email address'),
    role: z.enum(['admin', 'user', 'viewer'], {
      errorMap: () => ({ message: 'Please select a valid role' }),
    }),
    password: z
      .string()
      .min(8, 'Minimum 8 characters')
      .regex(/[A-Z]/, 'Must contain at least one uppercase letter')
      .regex(/[0-9]/, 'Must contain at least one number'),
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: 'Passwords do not match',
    path: ['confirmPassword'], // attach the error to the correct field
  })

type CreateUserFormValues = z.infer<typeof CreateUserFormSchema>

export const CreateUserForm = ({ onSuccess }: { onSuccess: () => void }) => {
  const { mutate: createUser, isPending, isError } = useCreateUser()

  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    reset,
    setError,
  } = useForm<CreateUserFormValues>({
    resolver: zodResolver(CreateUserFormSchema),
    defaultValues: {
      name: '',
      email: '',
      role: 'user',
      password: '',
      confirmPassword: '',
    },
  })

  const onSubmit = (data: CreateUserFormValues) => {
    createUser(
      { name: data.name, email: data.email, role: data.role },
      {
        onSuccess: () => {
          reset()
          onSuccess()
        },
        onError: (err) => {
          // Map API errors back to specific form fields
          if (err.response?.data?.field === 'email') {
            setError('email', { message: 'This email is already registered' })
          }
        },
      }
    )
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      {/* Each field: label + input + accessible error message */}
      <div>
        <label htmlFor="name">Full name</label>
        <input
          id="name"
          type="text"
          aria-describedby={errors.name ? 'name-error' : undefined}
          aria-invalid={Boolean(errors.name)}
          {...register('name')}
        />
        {errors.name && (
          <p id="name-error" role="alert">
            {errors.name.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email address</label>
        <input
          id="email"
          type="email"
          aria-describedby={errors.email ? 'email-error' : undefined}
          aria-invalid={Boolean(errors.email)}
          {...register('email')}
        />
        {errors.email && (
          <p id="email-error" role="alert">
            {errors.email.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="role">Role</label>
        <select id="role" {...register('role')}>
          <option value="user">User</option>
          <option value="admin">Administrator</option>
          <option value="viewer">Viewer</option>
        </select>
        {errors.role && <p role="alert">{errors.role.message}</p>}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          aria-describedby={errors.password ? 'password-error' : undefined}
          aria-invalid={Boolean(errors.password)}
          {...register('password')}
        />
        {errors.password && (
          <p id="password-error" role="alert">
            {errors.password.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="confirmPassword">Confirm password</label>
        <input
          id="confirmPassword"
          type="password"
          aria-invalid={Boolean(errors.confirmPassword)}
          {...register('confirmPassword')}
        />
        {errors.confirmPassword && (
          <p role="alert">{errors.confirmPassword.message}</p>
        )}
      </div>

      {/* Global API error */}
      {isError && (
        <div role="alert" aria-live="assertive">
          Failed to create user. Please try again.
        </div>
      )}

      <button type="submit" disabled={isSubmitting || isPending}>
        {isPending ? 'Creating...' : 'Create user'}
      </button>
    </form>
  )
}
```

---

## Dynamic field arrays

```typescript
import { useFieldArray } from 'react-hook-form'

const OrderFormSchema = z.object({
  items: z
    .array(
      z.object({
        productId: z.string().uuid('Select a valid product'),
        quantity: z.number().int().min(1, 'Minimum 1').max(99, 'Maximum 99'),
      })
    )
    .min(1, 'Add at least one product'),
})

export const OrderForm = () => {
  const {
    control,
    register,
    formState: { errors },
  } = useForm({
    resolver: zodResolver(OrderFormSchema),
    defaultValues: { items: [{ productId: '', quantity: 1 }] },
  })

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  })

  return (
    <form>
      {fields.map((field, index) => (
        <div key={field.id}>
          <label htmlFor={`items.${index}.productId`}>Product</label>
          <input
            id={`items.${index}.productId`}
            {...register(`items.${index}.productId`)}
          />

          <label htmlFor={`items.${index}.quantity`}>Quantity</label>
          <input
            id={`items.${index}.quantity`}
            type="number"
            {...register(`items.${index}.quantity`, { valueAsNumber: true })}
          />

          {errors.items?.[index]?.quantity && (
            <p role="alert">{errors.items[index].quantity?.message}</p>
          )}

          {fields.length > 1 && (
            <button type="button" onClick={() => remove(index)}>
              Remove
            </button>
          )}
        </div>
      ))}

      <button
        type="button"
        onClick={() => append({ productId: '', quantity: 1 })}
      >
        Add item
      </button>
    </form>
  )
}
```

---

## Form tests — required

```typescript
// CreateUserForm.test.tsx
import { render, screen, waitFor } from '@/tests/utils/test-utils'
import userEvent from '@testing-library/user-event'
import { CreateUserForm } from './CreateUserForm'

describe('CreateUserForm', () => {
  const user = userEvent.setup()
  const onSuccess = vi.fn()

  beforeEach(() => {
    vi.clearAllMocks()
  })

  it('shows validation errors when submitting empty form', async () => {
    render(<CreateUserForm onSuccess={onSuccess} />)

    await user.click(screen.getByRole('button', { name: 'Create user' }))

    expect(await screen.findByText('Name is required')).toBeInTheDocument()
    expect(await screen.findByText('Email is required')).toBeInTheDocument()
  })

  it('shows error for invalid email format', async () => {
    render(<CreateUserForm onSuccess={onSuccess} />)

    await user.type(screen.getByLabelText('Email address'), 'not-an-email')
    await user.click(screen.getByRole('button', { name: 'Create user' }))

    expect(await screen.findByText('Invalid email address')).toBeInTheDocument()
  })

  it('shows error when passwords do not match', async () => {
    render(<CreateUserForm onSuccess={onSuccess} />)

    await user.type(screen.getByLabelText('Password'), 'Password1')
    await user.type(screen.getByLabelText('Confirm password'), 'Different1')
    await user.click(screen.getByRole('button', { name: 'Create user' }))

    expect(await screen.findByText('Passwords do not match')).toBeInTheDocument()
  })

  it('calls onSuccess after successful form submission', async () => {
    render(<CreateUserForm onSuccess={onSuccess} />)

    await user.type(screen.getByLabelText('Full name'), 'John Doe')
    await user.type(screen.getByLabelText('Email address'), 'john@example.com')
    await user.type(screen.getByLabelText('Password'), 'Password1')
    await user.type(screen.getByLabelText('Confirm password'), 'Password1')
    await user.click(screen.getByRole('button', { name: 'Create user' }))

    await waitFor(() => {
      expect(onSuccess).toHaveBeenCalledTimes(1)
    })
  })

  it('disables submit button while request is pending', async () => {
    render(<CreateUserForm onSuccess={onSuccess} />)

    // Fill valid data and submit
    await user.type(screen.getByLabelText('Full name'), 'John Doe')
    await user.type(screen.getByLabelText('Email address'), 'john@example.com')
    await user.type(screen.getByLabelText('Password'), 'Password1')
    await user.type(screen.getByLabelText('Confirm password'), 'Password1')
    await user.click(screen.getByRole('button', { name: 'Create user' }))

    // Button should be disabled with loading text during the request
    expect(screen.getByRole('button', { name: 'Creating...' })).toBeDisabled()
  })
})
```

---

## Accessibility rules for forms

```
✅ Always:
  - <label htmlFor="id"> linked to <input id="id">
  - aria-describedby pointing to the error element ID
  - aria-invalid={true} when the field has an error
  - role="alert" on error messages (announces to screen readers)
  - aria-live="assertive" on global form-level error banners
  - Explicit type="submit" on submit buttons
  - Explicit type="button" on buttons that do not submit the form
  - disabled on submit button while isPending is true

❌ Never:
  - Placeholder text as the only label for a field
  - Error messages without association to their field
  - Forms with no loading state during submission
  - Using color alone to indicate an error (use text too)
```
