---
title: "Writing a custom notary service (experimental)"
date: 2020-01-08T09:59:25Z
---


# Writing a custom notary service (experimental)

{{< warning >}}
Customising a notary service is still an experimental feature and not recommended for most use-cases. The APIs
                for writing a custom notary may change in the future.


{{< /warning >}}
The first step is to create a service class in your CorDapp that extends the `NotaryService` abstract class.
            This will ensure that it is recognised as a notary service.
            The custom notary service class should provide a constructor with two parameters of types `ServiceHubInternal` and `PublicKey`.
            Note that `ServiceHubInternal` does not provide any API stability guarantees.

```kotlin
class MyCustomValidatingNotaryService(
        override val services: ServiceHubInternal,
        override val notaryIdentityKey: PublicKey)
    : SinglePartyNotaryService() {
    override val uniquenessProvider = PersistentUniquenessProvider(
            services.clock,
            services.database,
            services.cacheFactory,
            ::signTransaction)

    override fun createServiceFlow(otherPartySession: FlowSession): FlowLogic<Void?> = MyValidatingNotaryFlow(otherPartySession, this)

    override fun start() {}
    override fun stop() {}
}

```

{{/* /en/docs/corda-os/4.4/samples/notary-demo/workflows/src/main/kotlin/net/corda/notarydemo/MyCustomNotaryService.kt */}}
The next step is to write a notary service flow. You are free to copy and modify the existing built-in flows such
            as `ValidatingNotaryFlow`, `NonValidatingNotaryFlow`, or implement your own from scratch (following the
            `NotaryFlow.Service` template). Below is an example of a custom flow for a *validating* notary service:

```kotlin
class MyValidatingNotaryFlow(otherSide: FlowSession, service: MyCustomValidatingNotaryService) : ValidatingNotaryFlow(otherSide, service) {
    override fun verifyTransaction(requestPayload: NotarisationPayload) {
        try {
            val stx = requestPayload.signedTransaction
            resolveAndContractVerify(stx)
            verifySignatures(stx)
            customVerify(stx)
        } catch (e: Exception) {
            throw  NotaryInternalException(NotaryError.TransactionInvalid(e))
        }
    }

    @Suspendable
    private fun resolveAndContractVerify(stx: SignedTransaction) {
        subFlow(ResolveTransactionsFlow(stx, otherSideSession))
        stx.verify(serviceHub, false)
    }

    private fun verifySignatures(stx: SignedTransaction) {
        val transactionWithSignatures = stx.resolveTransactionWithSignatures(serviceHub)
        checkSignatures(transactionWithSignatures)
    }

    private fun checkSignatures(tx: TransactionWithSignatures) {
        tx.verifySignaturesExcept(service.notaryIdentityKey)
    }

    private fun customVerify(stx: SignedTransaction) {
        // Add custom verification logic
    }
}

```

{{/* /en/docs/corda-os/4.4/samples/notary-demo/workflows/src/main/kotlin/net/corda/notarydemo/MyCustomNotaryService.kt */}}
To enable the service, add the following to the node configuration:

```none
notary : {
    validating : true # Set to false if your service is non-validating
    className : "net.corda.notarydemo.MyCustomValidatingNotaryService" # The fully qualified name of your service class
}
```

## Testing your custom notary service
To create a flow test that uses your custom notary service, you can set the class name of the custom notary service as follows in your flow test:

```kotlin
        mockNet = MockNetwork(MockNetworkParameters(
                cordappsForAllNodes = listOf(DUMMY_CONTRACTS_CORDAPP, enclosedCordapp()),
                notarySpecs = listOf(MockNetworkNotarySpec(
                        name = CordaX500Name("Custom Notary", "Amsterdam", "NL"),
                        className = "net.corda.testing.node.CustomNotaryTest\$CustomNotaryService",
                        validating = false // Can also be validating if preferred.
                ))
        ))

```

{{/* /en/docs/corda-os/4.4/testing/node-driver/src/test/kotlin/net/corda/testing/node/CustomNotaryTest.kt */}}
After this, your custom notary will be the default notary on the mock network, and can be used in the same way as described in [Writing flow tests]({{< relref "flow-testing" >}}).

