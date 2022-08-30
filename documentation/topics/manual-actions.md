# Manual Actions

Manual actions are a way to implement an action in a fully custom way. This can be a very useful escape hatch when you have something that you are finding difficult to model with Ash's builtin tools.

## Manual Creates/Updates/Destroy

For manual create/update/destroy actions, everything works pretty much the same, with the exception that the `after_action` hooks on a resource will receive a `nil` value for creates, and the old unmodified value for updates, and you are expected to add an after action hook that changes that `nil` value into the result of the action.

For example:

# in the action

```elixir
create :special_create do
  manual? true
  change MyApp.DoCreate
end

# The change
defmodule MyApp.DoCreate do
  use Ash.Resource.Change

  def change(changeset, _, _) do
    Ash.Changeset.after_action(changeset, fn changeset, _result ->
      # result will be `nil`, because this is a manual action

      result = do_something_that_creates_the_record(changeset)

      {:ok, result}
    end)
  end
end
```

## Manual Read Actions

Manual read actions work differently. They must be provided a module that will run the read action.
The module should implement the `Ash.Resource.ManualRead` behaviour, and actions will simply be handed the ash query and the data layer query.

```elixir
# in the resource
actions do
  read :action_name do
    manual MyApp.ManualRead
    # or `{MyApp.ManualRead, ...opts}`
  end
end

# the implementation
defmodule MyApp.ManualRead do
  use Ash.Resource.ManualRead

  def read(ash_query, ecto_query, _opts, _context) do
    ...
    {:ok, query_results} | {:error, error}
  end
end
```

### Modifying the query

As an alternative to manual read actions, you can also provide the `modify_query` option, which takes an `MFA` and allows low level manipulation of the query just before it is dispatched to the data layer.

For example:

```elixir
read :read do
  modify_query {MyApp.ModifyQuery, :modify, []}
end

defmodule MyApp.ModifyQuery do
  def modify(ash_query, data_layer_query) do
    {:ok, modify_data_layer_query(data_layer_query)}
  end
end
```

This can be used as a last-resort escape hatch when you want to still use resource actions but need to do something that you can't do easily with Ash tools. As with any low level escape hatch, here be dragons.
