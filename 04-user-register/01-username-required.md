# username 必填

[上一章](/04-user-register/00-prepare.md)里，我们用 `mix phx.gen.html` 命令创建出完整用户界面，并且具备增加、删除、更改、查询用户的功能。

这一章，我们将实现 `username` 的第一个规则：`username` 必填，如果未填写，提示用户`请填写`。

先来看看，在 `http://localhost:4000/users/new` 页面上，提交空白用户名的话，我们会看到什么？

页面会提示我们，`can't be blank`。

很好，虽然不知道怎么回事，但**必填**的限制已经有了，那如何将它改成`请填写`呢？

打开 `web/models/user.ex` 文件，其中有一行：

```elixir
|> validate_required([:username, :email, :password])
```
正是 [`validate_required`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) 明确了 `username` 为必填。从文档里我们看到，`validate_required` 还接收一个可选的 `message` 参数，用于自定义错误消息。

让我们加上试试：

```elixir
diff --git a/tv_recipe/users/user.ex b/tv_recipe/users/user.ex
index b7713a0..87ce321 100644
--- a/web/models/user.ex
+++ b/web/models/user.ex
@@ -15,7 +15,7 @@ defmodule TvRecipe.User do
   def changeset(struct, params \\ %{}) do
     struct
     |> cast(params, [:username, :email, :password])
-    |> validate_required([:username, :email, :password])
+    |> validate_required([:username, :email, :password], message: "请填写")
     |> unique_constraint(:username)
     |> unique_constraint(:email)
   end
```

打开网址，提交空白用户名，页面上已经显示“请填写”了：

![show error when user submit blank username](/img/04-users-blank-username.png)

很好，但请注意，我们这是**人肉测试**。

又或者，我们可以用 Phoenix 生成的测试文件来验证。

打开 `test/tv_recipe/users_test.exs` 文件，默认内容如下：

```elixir
defmodule TvRecipe.UserTest do
  use TvRecipe.ModelCase

  alias TvRecipe.User

  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
  @invalid_attrs %{}

  test "changeset with valid attributes" do
    changeset = User.changeset(%User{}, @valid_attrs)
    assert changeset.valid?
  end

  test "changeset with invalid attributes" do
    changeset = User.changeset(%User{}, @invalid_attrs)
    refute changeset.valid?
  end
end
```
文件中有两个变量，`@valid_attrs` 表示有效的 `User` 属性，`@invalid_attrs` 表示无效的 `User` 属性，我们按本章开头拟定的规则修改 `@valid_attrs`：

```elixir
diff --git a/test/tv_recipe/users_test.exs b/test/tv_recipe/users_test.exs
index 1d5494f..7c73207 100644
--- a/test/tv_recipe/users_test.exs
+++ b/test/tv_recipe/users_test.exs
@@ -3,7 +3,7 @@ defmodule TvRecipe.UserTest do

   alias TvRecipe.User

-  @valid_attrs %{email: "some content", password: "some content", username: "some content"}
+  @valid_attrs %{email: "chenxsan@gmail.com", password: "some content", username: "chenxsan"}
   @invalid_attrs %{}

   test "changeset with valid attributes" do
```

接着，在 `users_test.exs` 文件中添加一个新测试：

```elixir
diff --git a/test/models/user_test.exs b/test/models/user_test.exs
index 7c73207..4c174ab 100644
--- a/test/tv_recipe/users_test.exs
+++ b/test/tv_recipe/users_test.exs
@@ -15,4 +15,9 @@ defmodule TvRecipe.UserTest do
     changeset = User.changeset(%User{}, @invalid_attrs)
     refute changeset.valid?
   end
+
+  test "username should not be blank" do
+    attrs = %{@valid_attrs | username: ""}
+    assert %{username: ["请填写"] } = errors_on(%User{}, attrs)
+  end
 end
```

这里，`%{@valid_attrs | username: ""}` 是 Elixir 更新映射（Map）的一个方法。

至于 `errors_on/2` 函数，它需要新增在 `test/support/data_case.ex` 文件中：

```elixir
   def errors_on(changeset) do
     Ecto.Changeset.traverse_errors(changeset, fn {message, opts} ->
       Regex.replace(~r"%{(\w+)}", message, fn _, key ->
         opts |> Keyword.get(String.to_existing_atom(key), key) |> to_string()
       end)
     end)
   end
+
+  def errors_on(struct, attrs) do
+    changeset = struct.__struct__.changeset(struct, attrs)
+    errors_on(changeset)
+  end
end
```

是否很吃惊？要知道，如果是在 JavaScript 里写两个同名函数，后一个函数会覆盖前一个的定义，而 Elixir 下，我们可以定义多个同名函数，它们能处理不同的状况，而又互不干扰。

它检查给定数据中的错误消息，并返回给我们。

现在在命令行下运行：

```bash
$ mix test test/tv_recipe/users_test.exs
```
结果如下：

```bash
...

Finished in 0.07 seconds
3 tests, 0 failures
```
测试通过。现在我们可以放心地认为，用户提交空白 `username` 时，Phoenix 一定会返回“请填写”的错误消息。

下一章，我们将[验证 username 的唯一性](/04-user-register/02-username-unique.md)。

## 为什么要写测试？

可能很多人都抱有这个疑问。测试增加了我们的工作量，而它的作用又不那么明显。更何况，这还只是一个入门教程。

我想谈几点个人感受：

1. 我不喜欢拿自己当人肉测试机。
2. 代码的修改是必然发生的，而我们在修改时无法保证周全，此时测试即任务清单，它帮我们指出，哪些地方的代码需统一修改，这样我们才能保证代码的质量。
3. 在团队协作中，你很难保证别人的代码不会破坏到自己的那部分。比如一个开源项目，有人在 github 上提了 pull request，你如果有测试代码，马上就能知道，这个 pull request 是否会破坏其它功能，如果你没有测试代码，好了，你得一行一行验证了 - 这种成本无论是对维护者还是贡献者来说，都是极大的浪费。

而况在 Phoenix 框架下，测试非常容易写。


上一章：[用户注册功能](/04-user-register/00-prepare.md)
下一章：[验证 username 的唯一性](/04-user-register/02-username-unique.md)

