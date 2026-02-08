import kotlinx.coroutines.*
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableSharedFlow
import kotlinx.coroutines.test.advanceTimeBy
import kotlinx.coroutines.test.advanceUntilIdle
import kotlinx.coroutines.test.runTest
import kotlin.test.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

/**
 * debounce function for suspending operations that cancels previous executions.
 */
suspend fun <T> Flow<T>.debounceLatest(
    timeoutMillis: Long,
    block: suspend (T) -> Unit
) = coroutineScope {
    var job: Job? = null

    collect { value ->
        job?.cancel()
        job = launch {
            try {
                delay(timeoutMillis)
                block(value)
            } catch (e: CancellationException) {
                throw e
            } catch (e: Throwable) {
                // optionally log
                throw e
            }
        }
    }
}

@OptIn(ExperimentalCoroutinesApi::class)
class DebounceLatestTest {

    @Test
    fun `only latest value runs after debounce window`() = runTest {
        val flow = MutableSharedFlow<Int>()
        val executed = mutableListOf<Int>()

        val job = launch {
            flow.debounceLatest(1000) {
                executed.add(it)
            }
        }

        flow.emit(1)
        advanceTimeBy(500)

        flow.emit(2)
        advanceTimeBy(500)

        flow.emit(3)
        advanceTimeBy(1000)

        advanceUntilIdle()

        assertEquals(listOf(3), executed)

        job.cancel()
    }

    @Test
    fun `cancels running block when new value arrives`() = runTest {
        val flow = MutableSharedFlow<Int>()
        val executed = mutableListOf<Int>()

        val job = launch {
            flow.debounceLatest(500) {
                delay(1000) // long-running work
                executed.add(it)
            }
        }

        flow.emit(1)
        advanceTimeBy(500) // debounce passes -> 1 starts

        advanceTimeBy(200) // still running
        flow.emit(2)   // cancels 1

        advanceTimeBy(500) // debounce for 2
        advanceTimeBy(1000)

        advanceUntilIdle()

        assertEquals(listOf(2), executed)

        job.cancel()
    }

    @Test
    fun `does not execute if cancelled before debounce`() = runTest {
        val flow = MutableSharedFlow<Int>()
        val executed = mutableListOf<Int>()

        val job = launch {
            flow.debounceLatest(1000) {
                executed.add(it)
            }
        }

        flow.emit(1)
        job.cancel() // cancel before debounce delay

        advanceTimeBy(2000)
        advanceUntilIdle()

        assertTrue(executed.isEmpty())
    }

    @Test
    fun `last pending debounce is cancelled on coroutine cancellation`() = runTest {
        val flow = MutableSharedFlow<Int>()
        val executed = mutableListOf<Int>()

        val job = launch {
            flow.debounceLatest(1000) {
                executed.add(it)
            }
        }

        flow.emit(42)
        advanceTimeBy(500)

        job.cancel() // cancel before debounce
        advanceUntilIdle()

        assertTrue(executed.isEmpty())
    }
}
