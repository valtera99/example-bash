Codecov Bash Example
====================

| [https://codecov.io][1] | [@codecov][2] | [hello@codecov.io][3] |
| ----------------------- | ------------- | --------------------- |

This repository serves as an example on how to integrate with Codecov for Bash/Shell language.

See example coverage here: [![codecov.io](http://codecov.io/github/codecov/example-bash/coverage.svg?branch=master)](http://codecov.io/github/codecov/example-bash?branch=master)

## Basic Usage

Run your tests with [kcov][5] in order to create the necessary coverage
reports. For example:

```
kcov output_path script.sh
```

## travis-ci.org

Add to your `.travis.yml` file.
```yml
language: generic

sudo: required

addons:
  apt:
    packages:
      - libcurl4-openssl-dev
      - libelf-dev
      - libdw-dev
      - cmake

after_success: |
  wget https://github.com/SimonKagstrom/kcov/archive/master.tar.gz &&
  tar xzf master.tar.gz &&
  cd kcov-master &&
  mkdir build &&
  cd build &&
  cmake .. &&
  make &&
  sudo make install &&
  cd ../.. &&
  rm -rf kcov-master &&
  mkdir -p coverage &&
  kcov coverage script.sh &&
  bash <(curl -s https://codecov.io/bash)
```
Additional apt packages are [kcov][5] build dependencies.
[Codecov][1] is integrated by the following line in `after_success:`

```yml
bash <(curl -s https://codecov.io/bash)
```

## Private Repos

Add to your `.travis.yml` file.

```yml
env:
  global:
    - CODECOV_TOKEN=:uuid-repo-token
```

View source and learn more about [Codecov Global Uploader][4]

We are happy to help if you have any questions. Please contact email our Support at [support@codecov.io](mailto:support@codecov.io)

[1]: https://codecov.io/
[2]: https://twitter.com/codecov
[3]: mailto:hello@codecov.io
[4]: https://github.com/codecov/codecov-python
[5]: https://simonkagstrom.github.io/kcov









  static QueueFile createQueueFile(File folder, String name) throws IOException {
    createDirectory(folder);
    File file = new File(folder, name);
    try {
      return new QueueFile(file);
    } catch (IOException e) {
      //noinspection ResultOfMethodCallIgnored
      if (file.delete()) {
        return new QueueFile(file);
      } else {
        throw new IOException("Could not create queue file (" + name + ") in " + folder + ".");
      }
    }
  }
  static synchronized SegmentIntegration create(
      Context context,
      Client client,
      Cartographer cartographer,
      ExecutorService networkExecutor,
      Stats stats,
      Map<String, Boolean> bundledIntegrations,
      String tag,
      long flushIntervalInMillis,
      int flushQueueSize,
      Logger logger,
      Crypto crypto) {
    PayloadQueue payloadQueue;
    try {
      File folder = context.getDir("segment-disk-queue", Context.MODE_PRIVATE);
      QueueFile queueFile = createQueueFile(folder, tag);
      payloadQueue = new PayloadQueue.PersistentQueue(queueFile);
    } catch (IOException e) {
      logger.error(e, "Could not create disk queue. Falling back to memory queue.");
      payloadQueue = new PayloadQueue.MemoryQueue();
    }
    return new SegmentIntegration(
        context,
        client,
        cartographer,
        networkExecutor,
        payloadQueue,
        stats,
        bundledIntegrations,
        flushIntervalInMillis,
        flushQueueSize,
        logger,
        crypto);
  }
  SegmentIntegration(
      Context context,
      Client client,
      Cartographer cartographer,
      ExecutorService networkExecutor,
      PayloadQueue payloadQueue,
      Stats stats,
      Map<String, Boolean> bundledIntegrations,
      long flushIntervalInMillis,
      int flushQueueSize,
      Logger logger,
      Crypto crypto) {
    this.context = context;
    this.client = client;
    this.networkExecutor = networkExecutor;
    this.payloadQueue = payloadQueue;
    this.stats = stats;
    this.logger = logger;
    this.bundledIntegrations = bundledIntegrations;
    this.cartographer = cartographer;
    this.flushQueueSize = flushQueueSize;
    this.flushScheduler = Executors.newScheduledThreadPool(1, new AnalyticsThreadFactory());
    this.crypto = crypto;
    segmentThread = new HandlerThread(SEGMENT_THREAD_NAME, THREAD_PRIORITY_BACKGROUND);
    segmentThread.start();
    handler = new SegmentDispatcherHandler(segmentThread.getLooper(), this);
    long initialDelay = payloadQueue.size() >= flushQueueSize ? 0L : flushIntervalInMillis;
    flushScheduler.scheduleAtFixedRate(
        new Runnable() {
          @Override
          public void run() {
            flush();
          }
        },
        initialDelay,
        flushIntervalInMillis,
        TimeUnit.MILLISECONDS);
  }
  @Override
  public void identify(IdentifyPayload identify) {
    dispatchEnqueue(identify);
  }
  @Override
  public void group(GroupPayload group) {
    dispatchEnqueue(group);
  }
  @Override
  public void track(TrackPayload track) {
    dispatchEnqueue(track);
  }
  @Override
  public void alias(AliasPayload alias) {
    dispatchEnqueue(alias);
  }
  @Override
  public void screen(ScreenPayload screen) {
    dispatchEnqueue(screen);
  }
  private void dispatchEnqueue(BasePayload payload) {
    handler.sendMessage(handler.obtainMessage(SegmentDispatcherHandler.REQUEST_ENQUEUE, payload));
  }
  void performEnqueue(BasePayload original) {
    // Override any user provided values with anything that was bundled.
    // e.g. If user did Mixpanel: true and it was bundled, this would correctly override it with
    // false so that the server doesn't send that event as well.
    ValueMap providedIntegrations = original.integrations();
    LinkedHashMap<String, Object> combinedIntegrations =
        new LinkedHashMap<>(providedIntegrations.size() + bundledIntegrations.size());
    combinedIntegrations.putAll(providedIntegrations);
    combinedIntegrations.putAll(bundledIntegrations);
    combinedIntegrations.remove("Segment.io"); // don't include the Segment integration.
    // Make a copy of the payload so we don't mutate the original.
    ValueMap payload = new ValueMap();
    payload.putAll(original);
    payload.put("integrations", combinedIntegrations);
    if (payloadQueue.size() >= MAX_QUEUE_SIZE) {
      synchronized (flushLock) {
