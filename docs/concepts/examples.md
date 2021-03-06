# AST

## Argument count

Below is an example of a query that returns all functions in a repository with more than four arguments:

```yaml

query:
  import codelingo/vcs/git
  import codelingo/ast/go
  git.repo:
    owner == "username"
    host == "myvcsgithost.com"
    name == "myrepo"
    git.commit:
      sha == "HEAD"
      go.project:
        @review comment
        go.func_type(depth = any):
          go.field_list:
            child_count > 4
```

Lexicons get data into the CodeLingo Platform and provide a list of Facts to query that data. In the above example, the Git Lexicon finds and clones the "myrepo" repository from the "myvcsgithost.com" VCS host. The "myrepo" repository must be publicly accessible for the Git Lexicon to access it.

The CodeLingo Platform can be queried directly with the `$ lingo run search` command or via [Functions](actions.md) which use queries stored in Tenets.

## Matching a function name

```yaml
tenets:
- name: first-tenet
  actions:
    codelingo/docs:
      body: example doc
    codelingo/review:
      comment: This is a function, name 'writeMsg', but you probably knew that.
  query:
    import codelingo/ast/go
    @review comment
    go.func_decl(depth = any):
      name == "writeMsg"
```

This will find funcs named "writeMsg". Save and close the file, then run `lingo run review`. Try adding another func called "readMsg" and run a review. Only the "writeMsg" func should be highlighted. Now, update the Tenet to find all funcs that end in "Msg":

```yaml
  query:
    import codelingo/ast/go
    @review comment
    go.func_decl:
      name as funcName
      regex(/.*Msg$/, funcName)
```

## CSharp

Iterative code, such as the following, can be more safely expressed declaratively using LINQ. For example:

```csharp
decimal total = 0;
foreach (Account account in myAccounts) {
  if (account.Status == "active") {
  total += account.Balance;
  }
}
```

can be expressed with:

```csharp
decimal total = (from account in myAccounts
          where account.Status == "active"
          select account.Balance).Sum();
```

The CLQL to match this pattern should find all variables that are declared before a foreach statement, and are incremented within the loop. The Facts for incrementing inside a foreach loop, and declaring a variable can be generated in the IDE:

![C# example Generation](../img/cs_decl.png)

Note: the `csharp.variable_declarator` has the `identifier_token` field that can be used to identify the `total` variable, but it spans the whole third line, so the whole line must be selected to generate that Fact. Since other elements are within that line, many extra Facts are generated. This is largely a property of the C# parser used by the underlying [lexicon](CLQL.md#lexicons).

![C# example Generation](../img/cs_inc.png)

The generated code can be turned into a working query by combining the above queries under the same scope, removing extraneous Facts, and using a CLQL variable to ensure that the `csharp.identifier_name` and `csharp.variable_declarator` Facts refer to the same variable:

```yaml
csharp.method_declaration:
  csharp.block:
    csharp.local_declaration_statement:
      csharp.variable_declaration:
        @review.comment
        csharp.variable_declarator:
          identifier_token as varName
    csharp.for_each_statement:
      csharp.add_assignment_expression(depth = any):
        @review comment
        csharp.identifier_name:
          identifier_token as varName
```

<br />

## C++

The following Tenet asserts that functions should not return local objects by reference. When the function returns and the stack is unwrapped, that object will be destructed, and the reference will not point to anything.

The following query finds this bug by matching all functions that return a reference type, and declare the returned value inside the function body:

```
import codelingo/ast/cpp
@review.comment
cc.func_decl:
  cc.func_header:
    cc.return_type:
      cc.reference
  cc.block_stmt:
    cc.declaration_stmt:
      cc.variable:
        name as returnedReference
    cc.return_stmt:
      cc.variable:
        name as returnedReference
```


## CLQL vs StyleCop

CLQL, like StyleCop, can express C# style rules and use them to analyze a project, file, repository, or Pull Request. CLQL, like StyleCop can customize a set of predefined rules to determine how they should apply to a given project, and both can define custom rules.

StyleCop supports custom rules by providing a SourceAnalyzer class with CodeWalker methods. The rule author can iterate through elements of the document and raise violations when the code matches a certain pattern. 

CLQL can express all rules that can be expressed in StyleCop. By abstracting away the details of document walking, CLQL can express in 9 lines a rule that takes ~50 lines of StyleCop. In addition to requiring, on average, 5x less code to express these patterns, CLQL queries can be generated by selecting the code elements in an IDE.

CLQL is not limited to C# like StyleCop. CLQL can express logic about other domains of logic outside of the scope of StyleCop, like Version Control.

## Empty Block Statements

StyleCop can use a custom rule to raise a violation for all empty block statements:

```csharp
namespace Testing.EmptyBlockRule {
    using global::StyleCop;
    using global::styleCop.CSharp;

    [SourceAnalyzer(typeof(CsParser))]
    public class EmptyBlocks : SourceAnalyzer
    {
        public override void AnalyzeDocument(CodeDocument document)
        {
            CsDocument csdocument = (CsDocument)document;
            if (csdocument.RootElement != null &amp;&amp; !csdocument.RootElement.Generated)
            {
                csdocument.WalkDocument(
                    new CodeWalkerElementVisitor&lt;object&gt;(this.VisitElement),
                    null,
                    null);
            }
        }

        private bool VisitElement(CsElement element, CsElement parentElement, object context)
        {
            if (statement.StatementType == StatementType.Block && statement.ChildStatements.Count == 0)
            {
                this.AddViolation(parentElement, statement.LineNumber, "BlockStatementsShouldNotBeEmpty");
            }
        }


        private bool VisitStatement(Statement statement, Expression parentExpression, Statement parentStatement, CsElement parentElement, object context)
        {
            if (statement.StatementType == StatementType.Block && statement.ChildStatements.Count == 0)
            {
                this.AddViolation(parentElement, statement.LineNumber, "BlockStatementsShouldNotBeEmpty");
            }
        }
    }
}
```

```xml
<SourceAnalyzer Name="EmptyBlocks">
  <Description>
    Code blocks should not be empty.
  </Description>
  <Rules>
    <RuleGroup Name="Fun Rules You Will Love">
      <Rule Name="BlockStatementsShouldNotBeEmpty" CheckId="MY1000">
        <Context>A block statement should always contain child statements.</Context>
        <Description>Validates that the code does not contain any empty block statements.</Description>
      </Rule>
    </RuleGroup>
  </Rules>
</SourceAnalyzer>
```

The same rule can be expressed in CLQL as the following [Tenet](tenets.md):

```yaml
tenets:
  - name: "EmptyBlock"
    actions:
      codelingo/docs:
        title: "Validates that the code does not contain any empty block statements."
      codelingo/review:
        comment: This function block is empty.
    query:
      import codelingo/ast/cpp
      @review comment
      cs.block_stmt(depth = any):
        exclude:
          cs.element
```

The VisitStatement function contains the core logic of this StyleCop rule:

```csharp
private bool VisitStatement(Statement statement, Expression parentExpression, Statement parentStatement, CsElement parentElement, object context)
{
    if (statement.StatementType == StatementType.Block && statement.ChildStatements.Count == 0)
    {
        this.AddViolation(parentElement, statement.LineNumber, "BlockStatementsShouldNotBeEmpty");
    }
}
```

The VisitStatement method is run at every node of the AST tree, then a violation is added if the node is a block statement with no children.
In CLQL, the match statement expresses the logic of the query. Traversal is entirely abstracted away, and the Tenet author only needs to express the condition for a "rule violation":

```clql
cs.block_stmt:
  exclude:
    cs.element
```

The above query will match against any block statement that does not contain anything at all. `cs.element` matches all C# elements, and the `exclude` operator performs [exclusion](#exclude).

## Access Modifier Declaration

In this example, we'll exclude StyleCop's long setup and document traversal boilerplate and focus on the query, which raises a violation for all non-generated code that doesn't have a declared access modifier:

```csharp
private bool VisitElement(CsElement element, CsElement parentElement, object context)
{
    // Make sure this element is not generated.
    if (!element.Generated)
    {
        // Flag a violation if the element does not have an access modifier.
        if (!element.Declaration.AccessModifier)
        {
            this.AddViolation(element, "AccessModifiersMustBeDeclared");
        }
    }
}
```

As in the [empty block statements example below](#empty-block-statements), to express the pattern in CLQL, the Tenet author only needs to express conditions in the VisitElement body:

```yaml
cs.element:
  generated == "false"
  cs.declaration_stmt:
    access_modifier == "false"
```

The above query matches all C# elements that are not generated, whose declaration does not have an access modifier.


# Runtime

## Detecting Memory Leaks

In the example below we have a database manager class that wraps up a third party library we use to return connections to a database.

From past profiles of our application, we expect the function `getDBCon` to use less than 10MB of memory. If it uses more than this, we want to be notified.

We can do this with the following Tenet:

```yaml
csprof.session:
  csprof.exec:
      command == "./scripts/build.sh"
      args == "-o ./bin/program.exe"
  csprof.exec:
    command == "./bin/program.exe"
    args == "/host:127.0.0.1 /db:testing"
  cs.file:
    filename == "./db/manager.cs"
    @review comment
    cs.method:
      name == "getDBCon"
      csprof.exit:
        memory_mb >= 10
```
 
Sometime in the future we decide to update the underlying library to the latest version. After profiling our application again, the Tenet catches that multiple instances of the `getDBCon` function have used more than the allowed 10MB of memory.

As we iterate over the issues, we see a steady increase in the memory consumed by the `getDBCon` function. Knowing that this didn't happen with the older version of the library, we suspect a memory leak may have been introduced in the update and further investigation is required.

Note: CLQL is able to assist in pinpointing the source of memory leaks, but that is outside the scope of this use case.

<br />

## Detecting Race Conditions
In the example below we have a database manager class that we use to update and read user records.

Our application has a number of different workers that operate asynchronously, making calls to the database manager at any time.

We need to know if our database manager is handling the asynchronous calls correctly, so we write a Tenet below to catch potential race conditions between two functions used by the workers:


```yaml
csprof.session:
  csprof.exec:
    command == "./scripts/build.sh"
    args == "-o ./bin/program.exe"
  csprof.exec:
    command == "./bin/program.exe"
    args == "/host:127.0.0.1 /db:testing"
  cs.file:
    filename == "./db/manager.cs"
    cs.method:
      name == "updateUser"
      csprof.block_start:
        time as startUpdate
      csprof.block_exit:
        time as exitUpdate
    @review comment
    cs.method:
      name == "getUser"
      csprof.block_start:
        time > $startUpdate
      csprof.block_start:
        time < $exitUpdate
```

This query uses [variables](#variables). If the `getUser` function is called while an instance of the `updateUser` function is in progress, the `getUser` function must return after the `updateUser` function to prevent a dirty read from the database. An issue will be raised if this does not hold true.

<br />

## Detecting Deadlocks

In the example below, we have an application used for importing data into a database from a number of different sources asynchronously. The `importData` function is particularly resource heavy on our server due to the raw amount of data that needs to be processed. Knowing this, we decide to write a Tenet to catch any idle instances of the `importData` function:

```yaml
cs.session:
  csprof.exec:
    command == "./scripts/build.sh"
    args == "-o ./bin/program.exe"
  csprof.exec:
    command == "./bin/program.exe"
    args == "/host:127.0.0.1 /db:testing"
  cs.file:
    filename == "./db/manager.cs"
    @review comment
    cs.method:
      name == "importData"
      csprof.duration:
        time_min >= 4
        average_cpu_percent <= 1
        average_memory_mb <= 10
```

If an instance of the `importData` runs for more than 4 minutes with unusually low resource usage, an issue will be raised as the function is suspect of deadlock.
