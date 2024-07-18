# React Performance Optimization
Performance Optimization and Design Pattern Topics for Frontend Interviews

## React Dev Tools

- React Dev Tools is an extension created by the React team
- Provides two new tabs in DevTools called Components and Profiler

  ### Components
  
  - Check the props/state/hooks of the component
  - Edit the props/state from dev tools
  - Search for a component
  - Check the component's tree
  - Log a component's data in the console
  - Check subcomponents
 
  ### Profiler

  - Allows you to test the performance of your components and shows which components to focus on for improvements
  - Provides the following metrics:
    - Component list, which will show you the components rendered during a session
    - Commit chart, which will show you the list of commits during a session
    - Flame chart, which will show you the component list
    - Ranked chart, which will show you the component list in a ranked manner
    - Record why each component rendered while profiling

  ### References
  - [How to Use React Dev Tools – With Example Code and Videos](https://www.freecodecamp.org/news/how-to-use-react-dev-tools)

## Client side Rendering vs Server Side Rendering

- Client side rendering is ideal for applications that require rich interaction after initial load such as Single Page Applications(SPA) where SEO is not a primary concern
- Server side rendering is best for content-heavy sites where SEO and fast initial load times are crucial, such as ecommerce and news websites
  
    ### Client side rendering
  
    - Rich interactions
    - Reduced server load
    - Better client side caching
    - Slower initial load
    - Bad for SEO
    - Dependency on user's device
    
    ### Server side rendering
  
    - Faster initial page load
    - Better SEO
    - Consistent performance
    - Increased load on the server
    - Frequent server requests can increase latency
    - More complex to implement

## Code Splitting

## Error Boundaries

- [react-error-boundary](https://www.npmjs.com/package/react-error-boundary)

## Web Vitals

- [Web vitals](https://web.dev/articles/vitals)

## Virtualization

-  [react-window](https://www.npmjs.com/package/react-window)
