Background
==========
.. note:: *Still in draft.*

Some brief notes and pointers about TEEs and Intel SGX, mostly based on
`Intel SGX Explained`_ by Victor Costan and Srinivas Devadas
:cite:`cryptoeprint:2016:086`.

Here's an overview of SGX, directly taken from from `Intel SGX Explained`_
:cite:`cryptoeprint:2016:086`:

    Secure remote computation (Figure 1) is the problem of executing software
    on a remote computer owned and maintained by an untrusted party, with some
    integrity and confidentiality guarantees. In the general setting, secure
    remote computation is an unsolved problem. Fully Homomorphic Encryption
    :cite:`fhe` solves the problem for a limited family of computations, but
    has an impractical performance overhead :cite:`10.1145/2046660.2046682`.

    Intel’s Software Guard Extensions (SGX) is the latest iteration in a long
    line of trusted computing (Figure 2) designs, which aim to solve the
    secure remote computation problem by leveraging trusted hardware in the
    remote computer. The trusted hardware establishes a secure container, and
    the remote computation service user uploads the desired computation and
    data into the secure container. The trusted hardware protects the data’s
    confidentiality and integrity while the computation is being performed on
    it.

    SGX relies on software attestation, like its predecessors, the TPM [71]
    and TXT [70]. Attestation (Figure 3) proves to a user that she is
    communicating with a specific piece of software running in a secure
    container hosted by the trusted hardware. The proof is a cryptographic
    signature that certifies the hash of the secure container's contents. It
    follows that the remote computer's owner can load any software in a secure
    container, but the remote computation service user will refuse to load her
    data into a secure container whose contents' hash does not match the
    expected value.

    The remote computation service user verifies the attestation key used to
    produce the signature against an endorsement certificate created by the
    trusted hardware's manufacturer. The certificate states that the
    attestation key is only known to the trusted hardware, and only used for
    the purpose of attestation.

    SGX stands out from its predecessors by the amount of code covered by the
    attestation, which is in the Trusted Computing Base (TCB) for the system
    using hardware protection. The attestations produced by the original TPM
    design covered all the software running on a computer, and TXT
    attestations covered the code inside a VMX [181] virtual machine. In SGX,
    an enclave (secure container) only contains the private data in a
    computation, and the code that operates on it.

    An SGX-enabled processor protects the integrity and confidentiality of the
    computation inside an enclave by isolating the enclave's code and data
    from the outside environment, including the operating system and
    hypervisor, and hardware devices attached to the system bus. At the same
    time, the SGX model remains compatible with the traditional software
    layering in the Intel architecture, where the OS kernel and hypervisor
    manage the computer's resources.

    -- Intel SGX Explained :cite:`cryptoeprint:2016:086`

The goal of a Trusted Execution Environment is to have, at the chip or hardware level,
a secure container in which some code can be executed such that the surrounding
environment such as the operating system cannot tamper with the code and its execution.
In the context of Intel SGX, this secure container is called an *enclave*.

A user wishing to compute some function in an enclave can gain trust in the enclave
via an attestation protocol. If the user of is performing the attestation from a
different computer than the one running the enclave code, then this attestation
protocol is referred to as a remote attestation, whereas a local attestation refers
when the attestation protocol is executed from an enclave hosted on the same hardware
than the enclave under attestation.

The trust anchor is the chip manufacturer as it holds a cryptographic key from which
multiple other cryptographic keys are derived, and which are crucial to the security
of the TEE.

Hardware
--------
Key material is burnt into efuses.

Measurement
-----------
The MRENCLAVE and MRSIGNER are known as measurements of an enclave. The
MRENCLAVE is also known as the enclave hash and can be used to verify that
an enclave contains the expected code and data. Enclave measurments are done
by SGX enabled processors but can also be simulated with a build toolchain,
and using the SgxSign tool from Intel's linux-sgx.

SIGSTRUCT
---------
From `Intel SGX Explained`_ by Victor Costan and Srinivas Devadas
:cite:`cryptoeprint:2016:086`:

.. image:: _static/sigstruct.png



Attestation
-----------
There are 2 types of attestation:

* _local_
* _remote_

Local attestation is when an enclave is attested by another enclave that
"sits" on the same CPU. Remote attestation is when a third party, that does
not need an SGX process requests a quote or report from a remote enclave, and
sends that report to Intel for verification.

In local attestation the verifier can verify the report without communicating
with Intel, whereas in remote attestation the verifier must contact Intel for
verifying the report.

In local attestation, the report contains a MAC tag which can verified by
another enclave that runs on the same CPU, as it has access to the secret key
that was used to create the MAC and which can be used to verify the MAC.

In remote attestation, the enclave being attested generates a local
attestation report which is verified by the Quoting Enclave (provided by
Intel). According to the Intel SGX explained paper, the quoting enclave
replaces the MAC of the report with a signature ... MUST check this in the
code. The quoting enclave signs the report with an attesation key, that it
obtains, encrypted, from a provisioning enclave. **The signature is
encrypted.**

Quote
^^^^^
**Structure of a quote:**

.. code-block:: cpp

    typedef struct _quote_t
    {
        uint16_t            version;        /* 0   */
        uint16_t            sign_type;      /* 2   */
        sgx_epid_group_id_t epid_group_id;  /* 4   */
        sgx_isv_svn_t       qe_svn;         /* 8   */
        sgx_isv_svn_t       pce_svn;        /* 10  */
        uint32_t            xeid;           /* 12  */
        sgx_basename_t      basename;       /* 16  */
        sgx_report_body_t   report_body;    /* 48  */
        uint32_t            signature_len;  /* 432 */
        uint8_t             signature[];    /* 436 */
    } sgx_quote_t;


(source: `linux-sgx/common/inc/sgx_quote.h <https://github.com/intel/linux-sgx/blob/bb3d1a5a302511954fcd1b20df4466554e129df1/common/inc/sgx_quote.h#L75-L87>`_)

EPID signature verification
"""""""""""""""""""""""""""
The Quoting Enclave, provided by Intel, is responsible for signing the quote,
and it encrypts the signature:

.. code-block:: cpp

    /* Encrypt the basic signature. */
    se_ret = sgx_aes_gcm128_enc_update(
        (uint8_t *)&basic_sig,   //start address to data before/after encryption
        sizeof(basic_sig),
        (uint8_t *)&encrypted_basic_sig, //length of data
        aes_gcm_state); //pointer to a state

(source `linux-sgx/psw/ae/qe/quoting_enclave.cpp <https://github.com/intel/linux-sgx/blob/bb3d1a5a302511954fcd1b20df4466554e129df1/psw/ae/qe/quoting_enclave.cpp#L536-L541>`_)

**Structure of signature:**

.. code-block:: cpp

    typedef struct _se_encrypted_sign
    {
        se_wrap_key_t       wrap_key;               /* 0 */
        uint8_t             iv[QUOTE_IV_SIZE];      /* 288 */
        uint32_t            payload_size;           /* 300 */
        BasicSignature      basic_sign;             /* 304, this field is encrypted, and contributes to the mac */
        uint32_t            rl_ver;                 /* 656, this field is encrypted, and contributes to the mac */
        uint32_t            rl_num;                 /* 660, this field is encrypted, and contributes to the mac */
        uint8_t             nrp_mac[];              /* 664, this filed contains the encrypted nrps followed by the mac */
    }se_encrypted_sign_t;

(source: `linux-sgx/common/inc/internal/se_quote_internal.h <https://github.com/intel/linux-sgx/blob/bb3d1a5a302511954fcd1b20df4466554e129df1/common/inc/internal/se_quote_internal.h#L50-L60>`_)


.. code-block:: cpp

    /// Intel(R) EPID 2.0 basic signature.
    /*!
     * Basic signature: (B, K, T, c, sx, sf, sa, sb)
     */
    typedef struct BasicSignature {
      G1ElemStr B;   ///< an element in G1
      G1ElemStr K;   ///< an element in G1
      G1ElemStr T;   ///< an element in G1
      FpElemStr c;   ///< an integer between [0, p-1]
      FpElemStr sx;  ///< an integer between [0, p-1]
      FpElemStr sf;  ///< an integer between [0, p-1]
      FpElemStr sa;  ///< an integer between [0, p-1]
      FpElemStr sb;  ///< an integer between [0, p-1]
    } BasicSignature;

source: `linux-sgx/external/epid-sdk/epid/common/types.h <https://github.com/intel/linux-sgx/blob/bb3d1a5a302511954fcd1b20df4466554e129df1/external/epid-sdk/epid/common/types.h#L220-L233>`_


The signature is encrypted, and consequently cannot be verified without Intel.

    Intel is not currently supporting 3rd party attestation verifications
    of EPID signatures for either Linkable or unlinkable.

    -- https://community.intel.com/t5/Intel-Software-Guard-Extensions/Verify-EPID-Signature/m-p/1085984#M706

Also see https://github.com/kudelskisecurity/sgxfun/blob/master/GETQUOTE.md.


To Trust or Not to Trust
------------------------
**How can one trust the output of an enclave?**

Assuming one trusts the physical security of a chip, that known attacks have
been mitigated, and that the enclave code is not vulnerable to side channel
attacks, then how can one be certain that the output of an enclave is
trustworthy? The short answer is:

    **audits** + **reproducible builds** + **remote attestation**

.. _audits:

Audits
^^^^^^
Audits are necessary to verify that the enclave code does indeed what it is
expected to do and that it meets specific security requirements. For instance,
it may be possible through a security audit to verify that the enclave was
implemented such that it is not vulnerable to certain side-channel attacks.
See https://arxiv.org/abs/2006.13598.

.. todo:: Provide references/citations.

It's essential to make sure that the source code being audited is the exact
code that was used to build the enclave (`Enclave.signed.so`) that is
deployed. Hence, a signed enclave binary must be reproducible from its source
code. The next section covers reproducible builds in the context of enclaves.

.. _reproducible-builds:

Reproducible builds
^^^^^^^^^^^^^^^^^^^
In the context of SGX enclaves, a reproducible build mainly
means that the MRENCLAVE remains constant.

.. _remote-attestation:

Remote attestation
^^^^^^^^^^^^^^^^^^
The remote attestation report also
contains the MRENCLAVE, and can therefore be checked against the source code,
and the pre-built enclave under audit. In other words, given a remote
attestation report, it's possible to verify that the report was generated
by an enclave binary, and it's possible to verify that the enclave binary
was built from a specific version of source code. Through this verification
process a user can then gain trust in the ``REPORT_DATA`` contained in the
remote attestation report. This ``REPORT_DATA`` can contain arbitrary data,
according to the needs of the application.


The auditee tool wishes to help a user of an application that relies on
some output of an enclave wishes

Current State & Motivation
--------------------------

Techincal Challenges
^^^^^^^^^^^^^^^^^^^^



.. _intel sgx explained: https://eprint.iacr.org/2016/086
