
# Is there a list of GHC extensions that are considered &apos;safe&apos;?

## Question
        
Occasionally, a piece of code I want to write isn't legal without at least one language extension. This is particularly true when trying to implement ideas in research papers, which tend to use whichever spiffy, super-extended version of GHC was available at the time the paper was written, without making it clear which extensions are actually required.

The result is that I often end up with something like this at the top of my .hs files:

    {-# LANGUAGE TypeFamilies
               , MultiParamTypeClasses
               , FunctionalDependencies
               , FlexibleContexts
               , FlexibleInstances
               , UndecidableInstances
               , OverlappingInstances #-}
    

I don't mind that, but often I feel as though I'm making blind sacrifices to appease the Great God of GHC. It complains that a certain piece of code isn't valid without language extension X, so I add a pragma for X. Then it demands that I enable Y, so I add a pragma for Y. By the time this finishes, I've enable three or four language extensions that I don't really understand, and I have no idea which ones are 'safe'.

To explain what I mean by 'safe':

*   I understand that `UndecidableInstances` is safe, because although it may cause the compiler to not terminate, as long as the code compiles it won't have unexpected side effects.
    
*   On the other hand, `OverlappingInstances` is clearly unsafe, because it makes it very easy for me to accidentally write code that gives runtime errors.
    

So my question is:

> Is there a list of GHCextensions which are considered 'safe' and which are 'unsafe'?

## Answer
        
It's probably best to look at what [SafeHaskell](http://hackage.haskell.org/trac/ghc/wiki/SafeHaskell) allows:

> **Safe Language**
> 
> The Safe Language (enabled through `-XSafe`) restricts things in two different ways:
> 
> 1.  Certain GHC LANGUAGE extensions are disallowed completely.
> 2.  Certain GHC LANGUAGE extensions are restricted in functionality.
> 
> Below is precisely what flags and extensions fall into each category:
> 
> *   Disallowed completely: `GeneralizedNewtypeDeriving`, `TemplateHaskell`
> *   Restricted functionality: `OverlappingInstances`, `ForeignFunctionInterface`, `RULES`, `Data.Typeable`
>     *   See Restricted Features below
> *   Doesn't Matter: all remaining flags.
> 
> **Restricted and Disabled GHC Haskell Features**
> 
> In the Safe language dialect we restrict the following Haskell language features:
> 
> *   `ForeignFunctionInterface`: This is mostly safe, but foreign import declarations that import a function with a non-IO type are be disallowed. All FFI imports must reside in the IO Monad.
> *   `RULES`: As they can change the behaviour of trusted code in unanticipated ways, violating semantic consistency they are restricted in function. Specifically any `RULES` defined in a module `M` compiled with `-XSafe` are dropped. `RULES` defined in trustworthy modules that `M` imports are still valid and will fire as usual.
> *   `OverlappingInstances`: This extension can be used to violate semantic consistency, because malicious code could redefine a type instance (by containing a more specific instance definition) in a way that changes the behaviour of code importing the untrusted module. The extension is not disabled for a module `M` compiled with `-XSafe` but restricted. While `M` can define overlapping instance declarations, they can only be used in `M`. If in a module `N` that imports `M`, at a call site that uses a type-class function there is a choice of which instance to use (i.e overlapping) and the most specific choice is from `M` (or any other Safe compiled module), then compilation will fail. It is irrelevant if module `N` is considered Safe, or Trustworthy or neither.
> *   `Data.Typeable`: We allow instances of `Data.Typeable` to be derived but we don't allow hand crafted instances. Derived instances are machine generated by GHC and should be perfectly safe but hand crafted ones can lie about their type and allow unsafe coercions between types. This is in the spirit of the original design of SYB.
> 
> In the Safe language dialect we disable completely the following Haskell language features:
> 
> *   `GeneralizedNewtypeDeriving`: It can be used to violate constructor access control, by allowing untrusted code to manipulate protected data types in ways the data type author did not intend. I.e can be used to break invariants of data structures.
> *   `TemplateHaskell`: Is particularly dangerous, as it can cause side effects even at compilation time and can be used to access abstract data types. It is very easy to break module boundaries with TH.

I recall having read that the interaction of `FunctionalDependencies` and `UndecidableInstances` can also be unsafe, because beyond allowing an unlimited context stack depth `UndecidableInstances` also lifts the so-called [coverage condition](http://www.haskell.org/ghc/docs/7.4.1/html/users_guide/type-class-extensions.html) (section 7.6.3.2), but I can't find a cite for this at the moment.

EDIT 2015-10-27: Ever since GHC gained support for type roles, `GeneralizedNewtypeDeriving` is no longer unsafe. (I'm not sure what else might have changed.)
