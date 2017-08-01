---
layout: post
title: Nativescript Button tap event in a Repeater Item
category: Nativescript
tags: [Nativescript]
---

This is a quick walkthrough to getting a buttons tap event working from within a repeater using Typescript and Nativescript.

_I am assuming that your page's bindingContext has already been bound to your viewmodel for this article_

First things first, add a button in your repeater item template:

{% raw %}
```xml
<Repeater items="{{ Products }}">
    <Repeater.itemsLayout>
        <StackLayout orientation="vertical" />
    </Repeater.itemsLayout>
    <Repeater.itemTemplate>
        <StackLayout orientation="horizontal">
            <Label text="{{ Name }}" width="80%" />
            <Button Text="Add" width="auto" tap="{{ $parents['Page'].AddItem }}" />
        </StackLayout>
    </Repeater.itemTemplate>
</Repeater>
```
{% endraw %}

This markup will display a list of products showing the product name and then a button to 'Add' to the basket for this example.

For the next part we need to define our 'AddItem' method (Your's will be different) in our Page's ViewModel:

```ts
import {Observable, EventData} from 'data/observable';
import {Product} from '../../shared/product';

export class ProductsViewModel extends Observable {

    ...
    
    public AddItem(args : EventData) : void {
    
        let product = <Product>(<Button>args.object).bindingContext;
        let _this = (<Button>args.object).page.bindingContext;
    
        _this.AddItemInternal(product, 1);
    }

    ...
}
```

You may notice here or have discovered yourself previously that you cannot use 'this' in the event handler, this is because this is bound to our Button when we fire the event.

In order for us to get our ViewModel instance we have to access this through the button and page objects (I have my viewmodel bound to the page).

And that's all.

There is one thing to note however, and that is that you do get some 'errors' show up in the tns CLI, this does not seem to affect the functionatlity but does warrent more investigation:

```
JS: Binding: Property: 'AddItem' is invalid or does not exist. SourceProperty: '$parents['Page'].AddItem'
```