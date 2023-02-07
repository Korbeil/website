+++
title = "Batching Symfony Messenger messages"
date = "2020-05-11"
tags = ["php", "symfony", "messenger"]
+++

Disclaimer: This post is a copy of my [JoliCode blog post](https://jolicode.com/blog/batching-symfony-messenger-messages).

Sometimes we want to make messages in Symfony Messenger that are consumed as a batch and not one by one. Recently we came across a situation where we send updated translations for our entities through Messenger and then send them to our translation provider.

But since there is a strong rate limitation on this translation provider, we can’t send them one by one. We must send them to _store_ all received ones by a given consumer and send all the messages if we wait more than **10 seconds without a new message or if we have more than 100 messages** stored.

Now let’s show how we made it works:

```php
// Symfony Messenger Message:
class TranslationUpdate
{
    public function __construct(
        public string $locale,
        public string $key,
        public string $value,
    ) {
    }
}
```

```php
class TranslationUpdateHandler implements MessageHandlerInterface
{
    private const BUFFER_TIMER = 10; // in seconds
    private const BUFFER_LIMIT = 100;
    private array $buffer = [];

    public function __construct(
        private MessageBusInterface $messageBus,
    ) {
        pcntl_async_signals(true);
        pcntl_signal(SIGALRM, \Closure::fromCallable([$this, 'batchBuffer']));
    }

    public function __invoke(TranslationUpdate $message): void
    {
        $this->buffer[] = $message;

        if (\count($this->buffer) >= self::BUFFER_LIMIT) {
            $this->batchBuffer();
        } else {
            pcntl_alarm(self::BUFFER_TIMER);
        }
    }

    private function batchBuffer(): void
    {
        if (0 === \count($this->buffer)) {
            return;
        }

        $translationBatch = new TranslationBatch($this->buffer);
        $this->messageBus->dispatch($translationBatch);
        $this->buffer = [];

    }
}
```

Here we can see the Messenger message, basically we dispatch it every time, we have a translation update but this could apply to any message we need.

Then we have the message handler which will get all the messages and put them in an array buffer. When the buffer hits 100 elements or if we have no new elements during 10s we will trigger the `batchBuffer` method.

For the 10s timer, we are using [`pcntl_alarm`](https://www.php.net/manual/en/function.pcntl-alarm.php) which allows us to make asynchronous call of the `batchBuffer` method if needed.

PCNTL is a way to handle system signals in our PHP code, you can read more about it in  [the PHP documentation](https://www.php.net/manual/en/intro.pcntl.php) or if you can read french [we made a blog post about it](https://jolicode.com/blog/les-signaux-posix-et-php). We will set a timer that will send a SIGALRM signal to the process after the given number of seconds. Then when the signal is received by the process it will call the callback function we gave as the second argument of [`pcntl_signal`](https://www.php.net/manual/en/function.pcntl-signal.php). The callback is set for our entire application, so we can use this batching trick **only once**.

Then in the `batchBuffer` method we are using a new Messenger dispatch because we want to keep track of the messages if there are issues, and if we hit the method with pcntl we won’t have Messenger retry handling if there is an exception.

```php
class TranslationBatch
{
    /**
     * @param TranslationUpdate[] $notifications
     */
    public function __construct(
        private array $notifications,
    ) {
    }
}
```

```php
class TranslationBatchHandler implements MessageHandlerInterface
{
    public function __invoke(TranslationBatch $message): void
    {
      // handle all our messages
    }
}
```

And finally we have the batch handler which will always get a list of messages to send!
With that we can batch our Messenger messages with ease and avoid using a cron!

Edit: This method is only a proof of work, if you want to use it in production I recommend to use some more persistent storage for the buffer like Redis.
