# Abstract Service Locator

This is an abstract service locator with no concrete implementation, used to facilitate dependency injection when constructors are unavailable. It was created for use with Unity applications, but it can be applied to any development environment that prohibits constructor based dependency injection.

In Unity, code is generally added to MonoBehaviour components. MonoBehaviour scripts do not support constructor based dependency injection.

The Service Locator pattern is often considered an antipattern, to be avoided. However, arguments against using a service locator typically point to constructor based dependency injection as the appropriate alternative. In an environment where constructor based dependency injection is impossible, many arguments against the Service Locator pattern become invalid.

### Installation

This package can be installed via Unity's Package Manager. The Package Manager is available in Unity version 2018.3 and later.

- Open the Package Manager window.
- Open the Add (+) menu in the Package Manager’s toolbar.
- Select the "Install package from git URL" button.
- Enter the URL: https://github.com/moonymachine/abstractservicelocator.git
- Select Install.

### Usage

Use `Locator.Get<T>()` or `Locator.TryGet<T>(out T)` to resolve dependencies in your MonoBehaviour components.

```csharp
using AbstractServiceLocator;
using UnityEngine;

public class ExampleMonoBehaviour : MonoBehaviour
{
	private IService Service;
	private ILog Logger;

	private void Awake()
	{
		Logger = Locator.Get<ILog>();
	}

	private void OnEnable()
	{
		if(!Locator.TryGet(out Service))
		{
			Logger?.LogError("A required service is unavailable!");
			enabled = false;
			return;
		}

		Service.Foo();
	}
}
```

### Implementation

This package does not include a concrete implementation of the `IServiceLocator` interface. This minimizes tight coupling to the static `Locator` in this package. Here is an example of how you can inject an actual service locator implementation for Unity.

```csharp
using System;
using AbstractServiceLocator;
using UnityEngine;

public static class CompositionRoot
{
	private static ServiceLocator ServiceLocator;
	private static IServiceLocator GetServiceLocator() => ServiceLocator;

	[RuntimeInitializeOnLoadMethod(RuntimeInitializeLoadType.BeforeSceneLoad)]
	private static void Initialize()
	{
		try
		{
			// Initialize a concrete implementation of the abstract IServiceLocator.
			ServiceLocator = ServiceLocator.Initialize();

			// Inject your IServiceLocator into the static Locator.
			Locator.Register(GetServiceLocator);
		}
		catch(Exception exception)
		{
			Debug.LogException(exception);
		}

		// Schedule Shut Down
		GameObject gameObject = new GameObject(nameof(Destructor));
		UnityEngine.Object.DontDestroyOnLoad(gameObject);
		gameObject.hideFlags = HideFlags.HideInHierarchy;
		Destructor destructor = gameObject.AddComponent<Destructor>();
		destructor.Destroyed += ShutDown;
	}

	private static void ShutDown()
	{
		Locator.Remove(GetServiceLocator);

		if(ServiceLocator != null)
		{
			try
			{
				// Shut Down
				ServiceLocator.ShutDown();
			}
			catch(Exception exception)
			{
				Debug.LogException(exception);
			}
			finally
			{
				ServiceLocator = null;
			}
		}
	}
}
```

```csharp
using System;
using UnityEngine;

// Check Script Execution Order in Project Settings to ensure this happens last.
[DefaultExecutionOrder(32000)]
public sealed class Destructor : MonoBehaviour
{
	public event Action Destroyed;

	private void OnDestroy()
	{
		if(Destroyed != null)
		{
			try
			{
				Destroyed();
			}
			catch(Exception exception)
			{
				Debug.LogException(exception);
			}
			finally
			{
				Destroyed = null;
			}
		}
	}
}
```
