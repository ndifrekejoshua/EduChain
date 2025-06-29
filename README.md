;; EduChain - Academic credential verification system

(define-non-fungible-token academic-credential uint)

;; Storage
(define-map credential-registry uint {institution: principal, student-name: (string-utf8 64), degree-program: (string-utf8 256), graduation-details: (string-utf8 256), verification-fee: uint})
(define-data-var credential-id-counter uint u0)

;; Error codes
(define-constant err-institution-only (err u500))
(define-constant err-credential-not-found (err u501))
(define-constant err-verification-failed (err u502))
(define-constant err-invalid-student-name (err u503))
(define-constant err-invalid-program (err u504))
(define-constant err-invalid-details (err u505))
(define-constant err-invalid-fee (err u506))
(define-constant err-invalid-credential-id (err u507))

;; Issue academic credential
(define-public (issue-credential (student-name (string-utf8 64)) (degree-program (string-utf8 256)) (graduation-details (string-utf8 256)) (verification-fee uint))
  (begin
    ;; Validate credential parameters
    (asserts! (> (len student-name) u0) err-invalid-student-name)
    (asserts! (> (len degree-program) u0) err-invalid-program)
    (asserts! (> (len graduation-details) u0) err-invalid-details)
    (asserts! (> verification-fee u0) err-invalid-fee)
    
    (let
      ((credential-id (var-get credential-id-counter))
       (institution tx-sender))
      
      ;; Mint credential NFT
      (try! (nft-mint? academic-credential credential-id institution))
      
      ;; Register credential details
      (map-set credential-registry credential-id {institution: institution, student-name: student-name, degree-program: degree-program, graduation-details: graduation-details, verification-fee: verification-fee})
      
      ;; Increment credential counter
      (var-set credential-id-counter (+ credential-id u1))
      
      (ok credential-id))))

;; Verify credential
(define-public (verify-credential (credential-id uint))
  (begin
    ;; Validate credential ID
    (asserts! (< credential-id (var-get credential-id-counter)) err-invalid-credential-id)
    
    (let
      ((credential-data (unwrap! (map-get? credential-registry credential-id) err-credential-not-found))
       (fee (get verification-fee credential-data))
       (institution (get institution credential-data))
       (current-owner (unwrap! (nft-get-owner? academic-credential credential-id) err-credential-not-found)))
      
      ;; Check verifier has sufficient funds
      (asserts! (>= (stx-get-balance tx-sender) fee) err-verification-failed)
      
      ;; Transfer payment to institution
      (try! (stx-transfer? fee tx-sender institution))
      
      ;; Transfer credential to verifier
      (try! (nft-transfer? academic-credential credential-id current-owner tx-sender))
      
      (ok true))))

;; Get credential details
(define-read-only (get-credential-details (credential-id uint))
  (map-get? credential-registry credential-id))

;; Check credential ownership
(define-read-only (owns-credential (credential-id uint) (holder principal))
  (is-eq (some holder) (nft-get-owner? academic-credential credential-id)))

;; Get credential owner
(define-read-only (get-credential-owner (credential-id uint))
  (nft-get-owner? academic-credential credential-id))