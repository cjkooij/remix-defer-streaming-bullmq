# Stream Progress Updates with Remix using Defer, Suspense, and Server Sent Events

This is an example of how to use Remix's Defer feature in combination with an EventStream to stream progress updates to the client.

We have a form on the homepage that dispatches a long-running process to create a new JSON file. This will take about a minute to complete and will constantly update itself with its progress, from `{ progress: 0 }` to `{ progress: 100 }`.

When it's complete, with a progress of 100, the JSON file will contain a new property `img` that points to a URL.

In a more practical scenario, you could use this to track the progress of rendering an image or video. Maybe you're generating a gif from code, or using an AI model to generate an image.

There are two separate ideas here, working together to achieve the optimal UX.

## What is Defer

Defer is a feature of Remix that allows you to return an unresolved Promise from a loader. The page will server-side render without waiting for the promise to resolve, and then when it finally does, the client will re-render with the new data.

This is especially useful for data-heavy pages, such as dashboards with many async datapoints. We don't want to wait for all of that data to load before we can show the user the page, so we can use Defer to show the page immediately, and then load the data in the background.

## How do we use defer?

When the user navigates to `items/$hash`, (we redirect them there automatically upon dispatching the long-running process), we have a loader that continuously watches the `$hash.json` file to check its progress. The loader defers a promise that will resolve only when the json's progress has hit 100.

```ts
export async function loader({ params }: LoaderArgs) {
  if (!params.hash) return redirect("/")
  const pathname = path.join("public", "items", `${params.hash}.json`)

  const file = fs.readFileSync(pathname)
  if (!file) return redirect("/")

  const item = JSON.parse(file.toString())
  if (!item) return redirect("/")

  if (item.progress === 100) {
    return defer({
      promise: item,
    })
  }

  return defer({
    promise: new Promise((resolve) => {
      const interval = setInterval(() => {
        const file = fs.readFileSync(pathname)
        if (!file) return

        const item = JSON.parse(file.toString())
        if (!item) return

        if (item.progress === 100) {
          clearInterval(interval)
          resolve(item)
        }

        return
      })
    }),
  })
}
```

From a user's point of view, the page will load normally, but the browser's native loading spinner will continue for a minute until the promise resolves and the image appears on-screen.

This is good because it doesn't block the user from doing anything else on the page while they wait, and we could have some placeholder text that says something like "Rendering your image..." to let them know what's going on, but we can improve the UX by showing them exactly how far along the process is.

## Event Streams and Server Sent Events

When people talk about streaming, they're often talking about streaming video or audio. But we can also stream data, and that's what we're doing here.

Server Sent Events are a standard part of the web API, but most frameworks don't make it easy to use them.

No matter what technology you're using, server sent events work by having an endpoint that does not immediately close its connection, and which sends a content type of `text/event-stream`.

In Remix, we can use a resource route to make this endpoint, and our loader will return a stream that constant checks our JSON file for its progress.

```ts
export async function loader({ request, params }: LoaderArgs) {
  const hash = params.hash

  return eventStream(request.signal, function setup(send) {
    const interval = setInterval(() => {
      const file = fs.readFileSync(path.join("public", "items", `${hash}.json`))

      if (file.toString()) {
        const data = JSON.parse(file.toString())
        const progress = data.progress
        send({ event: "progress", data: String(progress) })

        if (progress === 100) {
          clearInterval(interval)
        }
      }
    }, 200)

    return function clear(timer: number) {
      clearInterval(interval)
      clearInterval(timer)
    }
  })
}
```

On the client, while we're waiting for our deferred promise to resolve, we can consume that stream to know how far along our process is.

```ts
const stream = useEventSource(`/items/${params.hash}/progress`, {
  event: "progress",
})
```

## Putting them together

We present deferred data by using React Suspense to conditionally show the content when it's ready. Suspense provides a fallback element to show when the data is not yet ready.

Normally a loading spinner would go here, but we can use that to show our streamed progress instead.

```tsx
export default function Index() {
  const data = useLoaderData()
  const params = useParams()
  const stream = useEventSource(`/items/${params.hash}/progress`, {
    event: "progress",
  })

  return (
    <div>
      <Suspense fallback={<span> {stream}% </span>}>
        <Await resolve={data.promise} errorElement={<p>Error loading img!</p>}>
          {(promise) => <img alt="" src={promise.img} />}
        </Await>
      </Suspense>
    </div>
  )
}
```
