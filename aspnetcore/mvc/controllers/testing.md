---
title: Test controller logic in ASP.NET Core
author: ardalis
description: Learn how to test controller logic in ASP.NET Core with Moq and xUnit.
ms.author: riande
ms.date: 08/22/2018
uid: mvc/controllers/testing
---
# Test controller logic in ASP.NET Core

By [Steve Smith](https://ardalis.com/)

[Controllers](xref:mvc/controllers/actions) play a central role in any ASP.NET Core MVC app. As such, you should have confidence that controllers behave as intended. Automated tests can provide you with this confidence and can detect errors before they reach production.

[View or download sample code](https://github.com/aspnet/Docs/tree/master/aspnetcore/mvc/controllers/testing/sample) ([how to download](xref:tutorials/index#how-to-download-a-sample))

## Unit tests of controller logic

[Unit tests](/dotnet/articles/core/testing/unit-testing-with-dotnet-test) involve testing a part of an app in isolation from its infrastructure and dependencies. When unit testing controller logic, only the contents of a single action are tested, not the behavior of its dependencies or of the framework itself. As you unit test controller actions, focus only on the controller's behavior. A controller unit test avoids scenarios such as [filters](xref:mvc/controllers/filters), [routing](xref:fundamentals/routing), and [model binding](xref:mvc/models/model-binding). However, unit tests don't detect issues in the interaction between components, which is the purpose of [integration tests](xref:test/integration-tests).

If you're writing custom filters and routes, unit test them in isolation, not as part of tests on a particular controller action.

> [!TIP]
> [Create and run unit tests with Visual Studio](/visualstudio/test/unit-test-your-code).

To demonstrate unit testing, review the following controller. The Home controller displays a list of brainstorming sessions and allows the creation of new brainstorming sessions with a POST request:

[!code-csharp[](testing/sample/TestingControllersSample/src/TestingControllersSample/Controllers/HomeController.cs?highlight=12,16,21,42,43)]

The controller follows the [Explicit Dependencies Principle](/dotnet/standard/modern-web-apps-azure-architecture/architectural-principles#explicit-dependencies), expecting dependency injection to provide it with an instance of `IBrainstormSessionRepository`. This makes it possible to test the controller with a mocked service using a mock object framework, such as [Moq](https://www.nuget.org/packages/Moq/). The `HTTP GET Index` method has no looping or branching and only calls one method. To test the `Index` method, verify that a <xref:Microsoft.AspNetCore.Mvc.ViewResult> is returned with a `ViewModel` from the repository's `List` method.

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/HomeControllerTests.cs?name=snippet_Index_ReturnsAViewResult_WithAListOfBrainstormSessions)]

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/HomeControllerTests.cs?name=snippet_GetTestSessions)]

The Home controller's `HTTP POST Index` method tests should verify:

* The action method returns a Bad Request `ViewResult` with the appropriate data when `ModelState.IsValid` is `false`.
* The `Add` method on the repository is called and a <xref:Microsoft.AspNetCore.Mvc.RedirectToActionResult> is returned with the correct arguments when `ModelState.IsValid` is `true`.

Invalid model state is tested by adding errors using <xref:Microsoft.AspNetCore.Mvc.ModelBinding.ModelStateDictionary.AddModelError*> as shown in the first test below:

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/HomeControllerTests.cs?name=snippet_ModelState_ValidOrInvalid&highlight=8,15-16,37-39)]

The first test confirms that when `ModelState` isn't valid, the same `ViewResult` is returned as for a GET request. The test doesn't attempt to pass in an invalid model. That approach doesn't work, since model binding isn't running (although an [integration test](xref:test/integration-tests) would use model binding). In this case, model binding isn't tested. These unit tests are only testing the code in the action method.

The second test verifies that when `ModelState` is valid:

* A new `BrainstormSession` is added (via the repository).
* The method returns a `RedirectToActionResult` with the expected properties.

Mocked calls that aren't called are normally ignored, but calling `Verifiable` at the end of the setup call allows it to be verified in the test. This is done with the call to `mockRepo.Verify`, which fails the test if the expected method wasn't called.

> [!NOTE]
> The Moq library used in this sample makes it possible to mix verifiable, or "strict", mocks with non-verifiable mocks (also called "loose" mocks or stubs). Learn more about [customizing Mock behavior with Moq](https://github.com/Moq/moq4/wiki/Quickstart#customizing-mock-behavior).

Another controller in the app displays information related to a particular brainstorming session. The controller includes logic to deal with invalid id values:

[!code-csharp[](testing/sample/TestingControllersSample/src/TestingControllersSample/Controllers/SessionController.cs?name=snippet1&highlight=12-15,17-21)]

The controller action has three cases to test, one for each `return` statement:

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/SessionControllerTests.cs?name=snippet1&highlight=11-13,30-31,48-52)]

The app exposes functionality as a web API on the `api/ideas` route:

* A list of ideas associated with a brainstorming session is returned by the `ForSession` method. The `ForSession` method returns a list of `IdeaDTO` types.
* The `Create` method adds new ideas to a session.

[!code-csharp[](testing/sample/TestingControllersSample/src/TestingControllersSample/Api/IdeasController.cs?name=snippet1&highlight=1,21)]

Avoid returning business domain entities directly via API calls:

* Domain entities often include more data than the client requires.
* Domain entities unnecessarily couple the app's internal domain model with the publicly exposed API.

Mapping between domain entities and the types returned to the client can be performed manually using:

* A LINQ `Select`, as the sample app uses.
* A library such as [AutoMapper](https://github.com/AutoMapper/AutoMapper).

The unit tests for the `Create` and `ForSession` API methods:

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/ApiIdeasControllerTests.cs?name=snippet_ApiIdeasControllerTests&highlight=2,7,13,17,30,34,60-65,69,82-83,87,99-103)]

To test the behavior of the method when the `ModelState` is invalid, add a model error to the controller as part of the test. Don't try to test model validation or model binding in unit tests&mdash;just test the action method's behavior when confronted with an invalid `ModelState`.

The second test depends on the repository returning `null`, so the mock repository is configured to return `null`. There's no need to create a test database (in memory or otherwise) and construct a query that returns this result. The test can be accomplished in a single statement, as the sample code illustrates.

The `Create_ReturnsNewlyCreatedIdeaForSession` test verifies that the repository's `Update` method is called. The mock is called with `Verifiable`, and the mocked repository's `Verify` method is called to confirm the verifiable method is executed. It's not a unit test responsibility to ensure that the `Update` method saved the data&mdash;that can be performed with an integration test.

::: moniker range=">= aspnetcore-2.1"

In ASP.NET Core 2.1 or later, [ActionResult&lt;T&gt; type](xref:web-api/action-return-types#actionresultt-type) (<xref:Microsoft.AspNetCore.Mvc.ActionResult`1>) enables you to return a type deriving from `ActionResult` or return a specific type. The sample app includes a method that returns a `List<IdeaDTO>` objects for a given session id:

[!code-csharp[](testing/sample/TestingControllersSample/src/TestingControllersSample/Api/IdeasController.cs?name=snippet_ActionResult)]

Two tests of the `ForSessionActionResult` controller are included in the `ApiIdeasControllerTests`. The first test confirms that the method returns an `ActionResult`, session, and idea for a valid session id:

* The `ActionResult` type is `ActionResult<List<IdeaDTO>>`.
* The value of the `ActionResult` is a `List<IdeaDTO>`.
* The first item in the list is a valid `IdeaDTO` matching the idea added to the mock session.

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/ApiIdeasControllerTests.cs?name=snippet_ActionResult&highlight=14-15,17)]

The second test confirms that the controller returns an `ActionResult` but not a nonexistent session for a bad session id:

* The `ActionResult` type is `ActionResult<List<IdeaDTO>>`.
* The `ActionResult.Result` is a <xref:Microsoft.AspNetCore.Mvc.NotFoundObjectResult>.

[!code-csharp[](testing/sample/TestingControllersSample/tests/TestingControllersSample.Tests/UnitTests/ApiIdeasControllerTests.cs?name=snippet_ActionResultNotFoundObjectResult&highlight=13-14)]

::: moniker-end

## Additional resources

* <xref:test/integration-tests>
* [Explicit Dependencies Principle](http://deviq.com/explicit-dependencies-principle/)
