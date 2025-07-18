
# Using `Phoenix.Component.form/1` Without a Changeset  
> And when you might still want one for validation

Phoenix 1.7+ introduced `Phoenix.Component.form/1`, which powers form rendering in LiveView and function components. Most guides use it with Ecto changesets, but you don’t need a changeset at all to use `.form`.

This article covers:

- Using `.form` with a plain map
- Validating forms manually (no Ecto)
- Using an in-memory changeset (no database) for validation

---

## ✅ When to Skip the Changeset

Use a plain map when:

- The form doesn't map to a database record
- You want minimal setup
- You don't need Ecto validations

---

## 🧪 Using `.form` Without a Changeset (With Manual Validation)

You can build a form using a **plain map** and validate manually in your LiveView.

### 1. Set up default state

```elixir
@default_form %{
  name: "",
  email: "",
  message: ""
}
```

```elixir
def mount(_params, _session, socket) do
  form = Phoenix.Component.to_form(@default_form, as: "contact")
  {:ok, assign(socket, form: form, errors: %{})}
end
```

---

### 2. Render the form in HEEx

```heex
<.form for={@form} as="contact" phx-submit="submit" phx-change="validate">
  <.input field={@form[:name]} type="text" label="Name" />
  <%= if @errors[:name], do: content_tag(:p, @errors[:name], class: "text-red-500 text-sm") %>

  <.input field={@form[:email]} type="email" label="Email" />
  <%= if @errors[:email], do: content_tag(:p, @errors[:email], class: "text-red-500 text-sm") %>

  <.input field={@form[:message]} type="textarea" label="Message" />
  <%= if @errors[:message], do: content_tag(:p, @errors[:message], class: "text-red-500 text-sm") %>

  <.button>Send</.button>
</.form>
```

---

### 3. Manual validation logic

```elixir
defp validate(params) do
  %{}
  |> maybe_error(:name, params["name"] == "", "Name is required")
  |> maybe_error(:email, params["email"] == "", "Email is required")
  |> maybe_error(:email, not String.contains?(params["email"] || "", "@"), "Invalid email")
  |> maybe_error(:message, params["message"] == "", "Message is required")
end

defp maybe_error(errors, _field, false, _msg), do: errors
defp maybe_error(errors, field, true, msg), do: Map.put(errors, field, msg)
```

---

### 4. Handle LiveView events

```elixir
def handle_event("validate", %{"contact" => params}, socket) do
  errors = validate(params)
  {:noreply, assign(socket, form: to_form(params, as: "contact"), errors: errors)}
end

def handle_event("submit", %{"contact" => params}, socket) do
  errors = validate(params)

  if map_size(errors) == 0 do
    # Success: send email, etc.
    {:noreply,
     socket
     |> put_flash(:info, "Message sent!")
     |> assign(form: to_form(@default_form, as: "contact"), errors: %{})}
  else
    {:noreply, assign(socket, form: to_form(params, as: "contact"), errors: errors)}
  end
end
```

---

### 🔄 Summary of the Validation Cycle

| Step             | What Happens                                      |
|------------------|--------------------------------------------------|
| `phx-change`     | Runs `validate/1`, updates errors live           |
| `phx-submit`     | Final validation before proceeding               |
| Error rendering  | Manually show errors from `@errors`              |

---

## ✅ When You Still Want a Changeset (No DB Access)

If you need proper validation (e.g. email format, length, etc.), but don’t want to use the database, you can use `embedded_schema`.

---

### 1. In-memory schema for validation

```elixir
defmodule ContactForm do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key false
  embedded_schema do
    field :name, :string
    field :email, :string
    field :message, :string
  end

  def changeset(params) do
    %__MODULE__{}
    |> cast(params, [:name, :email, :message])
    |> validate_required([:name, :email, :message])
    |> validate_format(:email, ~r/@/)
  end
end
```

---

### 2. LiveView usage with changeset

```elixir
def mount(_params, _session, socket) do
  changeset = ContactForm.changeset(%{})
  form = Phoenix.Component.to_form(changeset, as: "contact")
  {:ok, assign(socket, form: form)}
end
```

```elixir
def handle_event("submit", %{"contact" => params}, socket) do
  changeset = ContactForm.changeset(params)

  if changeset.valid? do
    # Process valid form
    {:noreply, socket |> put_flash(:info, "Message sent!")}
  else
    {:noreply, assign(socket, form: to_form(changeset, as: "contact"))}
  end
end
```

---

### 3. Rendering errors with changeset

```heex
<.input field={@form[:email]} type="email" />
<%= for err <- @form[:email].errors do %>
  <p class="text-red-500 text-sm"><%= err %></p>
<% end %>
```

---

## 🧵 Final Comparison

| Approach                 | Use case                                      |
|--------------------------|-----------------------------------------------|
| `to_form(map)`           | Quick forms, no validation needed             |
| Manual validation (map)  | Lightweight form, with custom validations     |
| `to_form(changeset)`     | You want Ecto validations without a database  |

Use what fits your needs — and don’t feel pressured to reach for Ecto when a plain map will do.
