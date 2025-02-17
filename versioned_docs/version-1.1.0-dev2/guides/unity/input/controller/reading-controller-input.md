---
id: controller-input-values
title: Controller Input Values
sidebar_position: 0.5
date: 1/28/2022
tags: [UnityController, Input]
keywords: [UnityController, Input]
---

Using Unity Input System, you can read Magic Leap 2's controller input directly using the `InputAction.ReadValue<T>()` method. View [Unity's Documentation](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/manual/index.html) to learn more about Unity Input System Unity's Input System.

:::tip
You do not have to wait for the controller to connect to read values from the `MagicLeapInputs.ControllersActions` class.
:::

## Example

The script below enables the controller's input and debugs its Position, Rotation, Trigger, and Bumper values when the controller is connected.



```csharp showLineNumbers
using UnityEngine;

public class ExampleClass : MonoBehaviour
{
    // This is was autogenerated and allows developers to create a dynamic
    // instance of an InputActionAsset which includes predefined action maps
    // that correspond to all of the Magic Leap 2's input.
    private MagicLeapInputs _magicLeapInputs;
    
    // This class is an Action Map and was autogenerated by the Unity Input
    // System and includes predefined bindings for the Magic Leap 2 Controller
    // Input Events.
    private MagicLeapInputs.ControllerActions _controllerActions;

    void Start()
    {
       //Initialize the MagicLeapInputs like you would Unity's default action map.
       _magicLeapInputs = new MagicLeapInputs();
       _magicLeapInputs.Enable();
       //Initialize the ControllerActions based off the Magic Leap Input
       _controllerActions = new MagicLeapInputs.ControllerActions(_magicLeapInputs);
    }

    void Update()
    {
        //Only debug the values if the controller is connected.
        if(_controllerActions.IsTracked.IsPressed()){
         //Read the input values directly
         Debug.Log("Position:" + _controllerActions.Position.ReadValue<Vector3>());
         Debug.Log("Rotation:" + _controllerActions.Rotation.ReadValue<Quaternion>());
         Debug.Log("Trigger:" + _controllerActions.Trigger.ReadValue<float>());
         Debug.Log("Bumper:" + _controllerActions.Bumper.IsPressed());
         Debug.Log("Touchpad1Position:" + _controllerActions.TouchpadPosition.ReadValue<Vector2>());
         Debug.Log("Touchpad1Force:" + _controllerActions.TouchpadForce.ReadValue<float>());
       }
    }
}
```

## See also

- [`UnityAPI/InputActionAsset`](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/api/UnityEngine.InputSystem.InputActionAsset.html)
  - An asset containing action maps and control schemes.
- [`UnityAPI/ActionMap`](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/api/UnityEngine.InputSystem.InputActionMap.html)
  - A mechanism for collecting a series of input actions and treating them as a group.
- [`UnityAPI/InputAction`](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/api/UnityEngine.InputSystem.InputAction.html)
  - A named input signal that can flexibly decide which input data to tap.
- [Unity Input System Quick Start Guide](https://docs.unity3d.com/Packages/com.unity.inputsystem@1.0/manual/QuickStartGuide.html)
  - Learn how to get started with the new Unity Input System.
  