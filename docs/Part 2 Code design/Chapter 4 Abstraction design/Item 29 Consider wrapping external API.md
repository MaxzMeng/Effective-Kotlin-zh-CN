## Item 29: Consider wrapping external API

It is risky to heavily use an API that might be unstable. Both when creators clarify that it is unstable, and when we do not trust those creators to keep it stable. Remembering that we need to adjust every use in case of inevitable API change, we should consider limiting uses and separate them from our logic as much as possible. This is why we often wrap potentially unstable external library APIs in our own project. This gives us a lot of freedom and stability:

- We are not afraid of API changes because we would only need to change a single usage inside the wrapper.
- We can adjust the API to our project style and logic.
- We can replace it with a different library in case of some problems with this one. 
- We can change the behavior of these objects if we need to (of course, do it responsibly). 

There are also counterarguments to this approach: 

- We need to define all those wrappers.
- Our internal API is internal, and developers need to learn it just for this project. 
- There are no courses teaching how our internal API works. We should also not expect answers on Stack Overflow. 

Knowing both sides, you need to decide which APIs should be wrapped. A good heuristics that tells us how stable a library is are version number and the number of users. Generally, the more users the library has, the more stable it is. Creators are more careful with changes when they know that their small change might require corrections in many projects. The riskiest libraries are new ones with small popularity. Use them wisely and consider wrapping them into your own classes and functions to control them internally.