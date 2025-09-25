package com.sarvatra.inst.ee;

import com.sarvatra.api.bridge.requests.JsonRequestWrapper;
import com.sarvatra.common.Config;
import com.sarvatra.common.httpClient.ApiClientConfig;
import com.sarvatra.common.httpClient.HttpHelper;
import com.sarvatra.common.httpClient.HttpRequestWrapper;
import com.sarvatra.common.httpClient.HttpResponseWrapper;
import org.apache.commons.codec.binary.Base64;
import org.apache.commons.io.FileUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.exception.ExceptionUtils;
import org.apache.http.conn.ConnectTimeoutException;
import org.bouncycastle.util.Arrays;
import org.jdom2.Attribute;
import org.jdom2.Document;
import org.jdom2.Element;
import org.jdom2.JDOMException;
import org.jdom2.input.SAXBuilder;
import org.jpos.core.Configuration;
import org.jpos.iso.ISOException;
import org.jpos.iso.ISOUtil;
import org.jpos.util.LogEvent;
import org.json.JSONObject;

import javax.crypto.BadPaddingException;
import javax.crypto.Cipher;
import javax.crypto.IllegalBlockSizeException;
import javax.crypto.NoSuchPaddingException;
import javax.crypto.spec.GCMParameterSpec;
import javax.crypto.spec.IvParameterSpec;
import javax.crypto.spec.SecretKeySpec;
import java.io.*;
import java.net.SocketTimeoutException;
import java.nio.charset.StandardCharsets;
import java.security.*;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.PKCS8EncodedKeySpec;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Properties;

public abstract class HttpCall {
    protected String sourceId;
    protected LogEvent evt;
    protected Configuration cfg;
    protected String sessionKey;
    protected JSONObject requestPayload;
    protected JSONObject responsePayload;
    protected boolean requestSuccess;
    protected String message;
    protected int statusCode;
    protected String urn;
    protected String url;
    private boolean connectionTimeout;
    private boolean readTimeout;
    private Long switchRefId;
    private Date switchTxnDate;
    protected String cbsIrc;
    protected String cbsIrcDetails;

    private static final String AES_ALGO = "AES";
    private static final String AES_CIPHER = "AES/GCM/NoPadding";
    private static final String ENCODING = "UTF-8";
    private static final int GCM_TAG_LENGTH = 128;
    private static final int GCM_IV_LENGTH = 12;
    private static final String RSA_ALGO = "RSA/ECB/OAEPPadding";
    private static final String SHA256 = "SHA256withRSA";

    protected String actualApiResponse;

    public HttpCall(String sourceId, LogEvent evt, String url, Configuration cfg) {
        this.evt = evt;
        this.cfg = cfg;
        this.url = url;
        this.sourceId = sourceId;
    }

    protected abstract JSONObject preparePayload();

    protected abstract void parseAPIResponse();

    public void process() {
        HttpResponseWrapper responseWrapper;
        try {
            actualApiResponse = null;
            generateUrn(sourceId);
            generateSessionKey();
            Config config = new Config(getProperties());
            ApiClientConfig apiConfig = new ApiClientConfig(cfg.get("http-client", "SBIEIS"), config);
            HttpHelper httpHelper = new HttpHelper(apiConfig);
            HttpRequestWrapper requestWrapper = getRequestWrapper();
            evt.addMessage("Bank aekKey - " + sessionKey);
            evt.addMessage("Bank Url is - " + url);
            evt.addMessage("Bank Http Parameters - " + apiConfig);

            responseWrapper = httpHelper.post(url, requestWrapper);
        } catch (Exception e) {
            responseWrapper = null;
            evt.addMessage(ExceptionUtils.getStackTrace(e));
        }

        try {
            parseResponse(responseWrapper);
        } catch (Exception e) {
            evt.addMessage(ExceptionUtils.getStackTrace(e));
        }
    }

    private Properties getProperties() {
        Properties properties = new Properties();
        for (String key : cfg.keySet()) {
            properties.put(key, cfg.get(key));
        }
        return properties;
    }

    private void generateSessionKey() {
        byte[] bytes = new byte[32]; // 32 bytes = 256 bits
        new SecureRandom().nextBytes(bytes);
        sessionKey = Base64.encodeBase64String(bytes);
    }

//    private byte[] getIVFromSessionKey() throws NoSuchAlgorithmException {
//        MessageDigest sha = MessageDigest.getInstance("SHA-256");
//        byte[] keyBytes = sha.digest(sessionKey.getBytes(StandardCharsets.UTF_8));
//        return Arrays.copyOf(keyBytes, 12);
//    }

    private String decrypt(final String encryptedText) throws Exception {
        byte[] encrypted = Base64.decodeBase64(encryptedText);
        byte[] keyBytes = Base64.decodeBase64(sessionKey);
        SecretKeySpec keySpec = new SecretKeySpec(keyBytes, AES_ALGO);
        IvParameterSpec ivTemp= new IvParameterSpec(Arrays.copyOf(sessionKey.getBytes(StandardCharsets.UTF_8),12));
        GCMParameterSpec iv =new GCMParameterSpec(GCM_TAG_LENGTH,ivTemp.getIV());
        Cipher cipher = Cipher.getInstance(AES_CIPHER);
        cipher.init(Cipher.DECRYPT_MODE, keySpec, iv);

        byte[] plain = cipher.doFinal(encrypted);
        return new String(plain, StandardCharsets.UTF_8);
    }

    private String encrypt(final String plainText) throws Exception {
        byte[] plainTextBytes = plainText.getBytes(ENCODING);
        byte[] keyBytes = Base64.decodeBase64(sessionKey);
        SecretKeySpec keySpec = new SecretKeySpec(keyBytes, "AES");
        IvParameterSpec ivTemp= new IvParameterSpec(Arrays.copyOf(keyBytes, 12));
        GCMParameterSpec iv =new GCMParameterSpec(128, ivTemp.getIV());
        Cipher cipher = Cipher.getInstance(AES_CIPHER);
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, iv);

        byte[] encrypt = cipher.doFinal(plainTextBytes);

        return Base64.encodeBase64String(encrypt);
    }

    private PublicKey getEisPublicKey() throws Exception {
        String certiPath = cfg.get("eis_public_key", "cfg/eis_public_key.pem");
        try (FileInputStream fin = new FileInputStream(certiPath)) {
            CertificateFactory f = CertificateFactory.getInstance("X.509");
            X509Certificate certificate = (X509Certificate) f.generateCertificate(fin);
            return certificate.getPublicKey();
        }
    }

    private String encryptRSA(String data, PublicKey publicKey) throws InvalidKeyException, NoSuchAlgorithmException, IllegalBlockSizeException, BadPaddingException, UnsupportedEncodingException, NoSuchPaddingException {
        Cipher cipher = Cipher.getInstance(RSA_ALGO);
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        byte[] cipherData = cipher.doFinal(data.getBytes(ENCODING));
        return Base64.encodeBase64String(cipherData);
    }

    private PrivateKey getSrvtPrivateKey() throws Exception {
        File file = new File(cfg.get("srvt_sbi_private_key", "cfg/srvt_sbi_private_key.pem"));
        String privateKeyString = FileUtils.readFileToString(file, StandardCharsets.UTF_8);

        String pk = privateKeyString
                .replaceAll("-----BEGIN (.*)-----", "")
                .replaceAll("-----END (.*)-----", "")
                .replaceAll("\\s", "");
        byte[] encoded = Base64.decodeBase64(pk);
        KeyFactory kf = KeyFactory.getInstance("RSA");
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(encoded);
        return kf.generatePrivate(keySpec);
    }

    private String getSignature(String data) throws Exception, InvalidKeySpecException {
        Signature signature = Signature.getInstance(SHA256);
        signature.initSign(getSrvtPrivateKey());
        signature.update(data.getBytes(ENCODING));
        return Base64.encodeBase64String(signature.sign());
    }

    protected boolean verifySignature(String data) throws Exception {
        Signature signature = Signature.getInstance(SHA256);
        signature.initVerify(getEisPublicKey());
        return signature.verify(Base64.decodeBase64(data));
    }

    protected JSONObject encryptPayload(String payload, String urn) throws Exception {
        String encryptPayloadStr = encrypt(payload);
        String signaturePayload = getSignature(payload);
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("REQUEST_REFERENCE_NUMBER", urn);
        jsonObject.put("REQUEST", encryptPayloadStr);
        jsonObject.put("DIGI_SIGN", signaturePayload);
        return jsonObject;
    }


    private HttpRequestWrapper<JSONObject> getRequestWrapper() throws Exception {
        requestPayload = preparePayload();
        evt.addMessage("Request plain payload - " + requestPayload);

        JsonRequestWrapper requestWrapper = new JsonRequestWrapper();

        HashMap<String, String> header = new HashMap<>();
        header.put("Content-Type", "application/json");
        header.put("AccessToken", getAccessToken());

        evt.addMessage("Request Header  - " + header);

        JSONObject encryptedPayload = encryptPayload(requestPayload.toString(), urn);
        evt.addMessage("Request encrypted payload - " + encryptedPayload);

        requestWrapper.setRequestBody(encryptedPayload);
        requestWrapper.setHeaders(header);
        return requestWrapper;
    }

    private void parseResponse(HttpResponseWrapper responseWrapper) throws Exception {
        if (responseWrapper == null) {
            requestSuccess = false;
            message = "Response is blank";
            return;
        }
        statusCode = responseWrapper.getHttpStatusCode();
        evt.addMessage("Status code - " + statusCode);

        if (statusCode != 200) {
            requestSuccess = false;
            message = "Error while executing request";
            if (responseWrapper.getCommunicationError() != null) {
                if (responseWrapper.getCommunicationError() instanceof ConnectTimeoutException) {
                    connectionTimeout = true;
                    message = "connection Timeout";
                } else if (responseWrapper.getCommunicationError() instanceof SocketTimeoutException) {
                    readTimeout = true;
                    message = "read timeout";
                }
            }
            return;
        }
        if (responseWrapper.getResponseBody() == null) {
            requestSuccess = false;
            message = "Request is successful however response is blank";
            return;
        }

        JSONObject encryptedPayload;
        try {
            encryptedPayload = new JSONObject(responseWrapper.getResponseBody());
        } catch (Exception e) {
            requestSuccess = false;
            message = "Response is not in json format";
            return;
        }
        evt.addMessage("Response encrypted payload - " + encryptedPayload);

        if (verifySignature(encryptedPayload.getString("DIGI_SIGN"))) {
            requestSuccess = false;
            message = "Invalid or missing DIGI_SIGN in response";
            return;
        }


        if (!encryptedPayload.has("RESPONSE")) {
            requestSuccess = false;
            message = "Request is failed at bank cbs";
            return;
        }
        String decryptedPayload;
        try {
            decryptedPayload = decryptPayload(encryptedPayload.getString("RESPONSE"));
        } catch (Exception e) {
            requestSuccess = false;
            message = "Unable to decrypt the response payload";
            return;
        }

        if (decryptedPayload == null) {
            requestSuccess = false;
            message = "Decrypted payload is null";
            return;
        }

        try {
            responsePayload = new JSONObject(decryptedPayload);
        } catch (Exception e) {
            requestSuccess = false;
            message = "Unable to convert response in json format";
            return;
        }


        evt.addMessage("Response plain payload - " + responsePayload);

        if (!responsePayload.has("RESPONSE_STATUS")) {
            requestSuccess = false;
            message = "RESPONSE_STATUS is not exist in response payload";
            return;
        }
        if (responsePayload != null && responsePayload.has("ERROR_CODE") && StringUtils.isNotBlank(responsePayload.getString("ERROR_CODE"))) {
            requestSuccess = false;
            cbsIrc = responsePayload.optString("ERROR_CODE");
            evt.addMessage("RC found : " + cbsIrc);
            cbsIrcDetails = responsePayload.optString("ERROR_DESCRIPTION");
            evt.addMessage("Description found : " + cbsIrcDetails);
            return;
        }

        actualApiResponse = responsePayload.getString("RESPONSE_STATUS");
        cbsIrc = responsePayload.getString("RESPONSE_STATUS");
        if (!StringUtils.equals(responsePayload.getString("RESPONSE_STATUS"), cfg.get("success-response-status", "0"))) {
            requestSuccess = false;
            message = "Transaction is failed";
            return;
        }
        parseAPIResponse();
    }

    public static String generateUrnumber(String sourceId, Date _switchTxnDate, Long _switchRefId) throws ISOException {
        SimpleDateFormat f = new SimpleDateFormat("yyDDDHHmmsss");
        String dateStr = (_switchTxnDate != null) ? f.format(_switchTxnDate) : f.format(new Date());

        int stan = (_switchRefId != null) ? _switchRefId.intValue() : new SecureRandom().nextInt();
        String randomStr = ISOUtil.zeropad(Integer.toString(Math.abs(stan) % 100000000), 8);
        return new StringBuilder("SBI").append(sourceId).append(dateStr).append(randomStr).toString();
    }
    private void generateUrn(String sourceId) throws ISOException {
        urn = generateUrnumber(sourceId, switchTxnDate, switchRefId);
    }

    protected String getAccessToken() throws Exception {
        return encryptRSA(sessionKey, getEisPublicKey());
    }

    protected String decryptPayload(String payload) throws Exception {
        return decrypt(payload);
    }

    public boolean isRequestSuccess() {
        return requestSuccess;
    }

    public String getActualApiResponse() {
        return actualApiResponse;
    }

    public String getMessage() {
        return message;
    }

    public String getUrn() {
        return urn;
    }

    protected String SHA256(String s) {
        try {
            if (StringUtils.isBlank(s)) return null;
            MessageDigest md = MessageDigest.getInstance("SHA-256");
            return Base64.encodeBase64String(md.digest(s.getBytes()));
        } catch (NoSuchAlgorithmException e) {
            evt.addMessage("SHA256 :" + ExceptionUtils.getStackTrace(e));
        }
        return s;
    }

    protected String generateTxnSecKey(String secretKey, String code, String rrn, String reqDt, String inputData) {
        StringBuilder strbuilder = new StringBuilder();
        strbuilder.append(secretKey).append("").append(code).append("").append(rrn)
                .append("").append(reqDt).append("").append(inputData);
        evt.addMessage("Before Sha256 -(redacted)");
        return SHA256(strbuilder.toString());
    }

    public Document getDocument(String bdy) throws IOException, JDOMException {
        SAXBuilder builder = new SAXBuilder();
        return builder.build(new ByteArrayInputStream(bdy.getBytes(StandardCharsets.UTF_8)));
    }

    public String getField(Document doc, String field) {
        Element msg = doc.getRootElement().getChild(field);
        return msg != null ? msg.getText() : null;
    }

    public String getAttributeValue(Document doc, String attribute) {
        Attribute att = doc.getRootElement().getAttribute(attribute);
        String value = null;
        if (att != null) {
            value = att.getValue();
        }
        return value != null ? value : null;
    }

    public boolean isConnectionTimeout() {
        return connectionTimeout;
    }

    public void setConnectionTimeout(boolean connectionTimeout) {
        this.connectionTimeout = connectionTimeout;
    }

    public boolean isReadTimeout() {
        return readTimeout;
    }

    public void setReadTimeout(boolean readTimeout) {
        this.readTimeout = readTimeout;
    }

    public Long getSwitchRefId() {
        return switchRefId;
    }

    public void setSwitchRefId(Long switchRefId) {
        this.switchRefId = switchRefId;
    }

    public Date getSwitchTxnDate() {
        return switchTxnDate;
    }

    public void setSwitchTxnDate(Date switchTxnDate) {
        this.switchTxnDate = switchTxnDate;
    }

    public String getCbsIrc() {
        return cbsIrc;
    }

    public String getCbsIrcDetails() {
        return responsePayload != null ? responsePayload.toString() : null ;
    }
}

RTSP DB Changes: 
New table is added in RTSP DB:
MIN_KYC_WHITELIST_DETAILS:
Create table Query:
CREATE TABLE "RTSP"."MIN_KYC_WHITELIST_DETAILS" (  CREATE TABLE “RTSP"ID" NUMBER(19,0) GENERATED BY DEFAULT AS IDENTITY MINVALUE 1 MAXVALUE 9999999999999999999999999999 INCREMENT BY 1 START WITH 1 CACHE 20 NOORDER  NOCYCLE  NOKEEP  NOSCALE  NOT NULL ENABLE, 
  "BENEFICIARY_NAME" VARCHAR2(1024 BYTE) DEFAULT NULL, 
  "MOBILE" VARCHAR2(64 BYTE) DEFAULT NULL NOT NULL ENABLE, 
  "AADHAR_LAST_FOUR_DIGITS" VARCHAR2(64 BYTE) DEFAULT NULL NOT NULL ENABLE, 
  "PROGRAM_NAME" VARCHAR2(256 BYTE) DEFAULT NULL, 
  "IS_VALID" CHAR(1 BYTE) DEFAULT 'N', 
  "BATCH" VARCHAR2(256 BYTE), 
  "CREATOR_ID" NUMBER(20,0), 
  "UPLOAD_DATE" DATE DEFAULT SYSDATE, 
  "FILE_NAME" VARCHAR2(255 BYTE), 
  "AADHAR_WALLET_AVAILABLE" VARCHAR2(5 BYTE), 
  "CREATOR_NAME" VARCHAR2(30 BYTE), 
  "ELIGIBLE_FOR_DISBURSEMENT" CHAR(1 BYTE) DEFAULT 'N', 
  "REMARKS" VARCHAR2(255 BYTE)
   ) SEGMENT CREATION IMMEDIATE 
  PCTFREE 10 PCTUSED 40 INITRANS 1 MAXTRANS 255 
  NOCOMPRESS LOGGING
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "RTSP_DATA" ;
Alter query for Primary key:
  ALTER TABLE "RTSP"."MIN_KYC_WHITELIST_DETAILS" ADD PRIMARY KEY ("ID")
  USING INDEX PCTFREE 10 INITRANS 2 MAXTRANS 255 COMPUTE STATISTICS 
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "RTSP_DATA" ENABLE;
Alter table for Constraint:
  ALTER TABLE "RTSP"."MIN_KYC_WHITELIST_DETAILS" ADD CONSTRAINT "CHK_IS_VALID" CHECK (IS_VALID IN ('Y', 'N')) ENABLE;
 Comments for columns:
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."ID" IS 'Primary Key';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."BENEFICIARY_NAME" IS 'Name of the Beneficiary';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."MOBILE" IS 'Mobile Number for Wallet Creation';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."AADHAR_LAST_FOUR_DIGITS" IS 'Last 4 Digits of Aadhar Card';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."PROGRAM_NAME" IS 'Disbursement Program Name';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."IS_VALID" IS 'Duplicate Check Y is Valid N is Invalid';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."BATCH" IS 'Batch ID for Upload Tracking';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."CREATOR_ID" IS 'Uploader Customer ID';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."UPLOAD_DATE" IS 'Date and time when the record was uploaded';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."FILE_NAME" IS 'Name of Uploaded File';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."AADHAR_WALLET_AVAILABLE" IS 'Y if Available else N';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."CREATOR_NAME" IS 'Name of the Uploaded';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."ELIGIBLE_FOR_DISBURSEMENT" IS 'Y if Eligible else N';
   COMMENT ON COLUMN "RTSP"."MIN_KYC_WHITELIST_DETAILS"."REMARKS" IS 'Remarks';
 Indexing for Columns:
 CREATE INDEX "RTSP"."IDX_MKYC_MOBILE_AADHAR_ELIGIBLE" ON "RTSP"."MIN_KYC_WHITELIST_DETAILS" ("MOBILE", "AADHAR_LAST_FOUR_DIGITS", "ELIGIBLE_FOR_DISBURSEMENT", "IS_VALID") 
  PCTFREE 10 INITRANS 2 MAXTRANS 255 COMPUTE STATISTICS 
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "RTSP_DATA";
 CREATE INDEX "RTSP"."MIN_KYC_WHITELIST_DETAILS_IDX" ON "RTSP"."MIN_KYC_WHITELIST_DETAILS" ("MOBILE") 
  PCTFREE 10 INITRANS 2 MAXTRANS 255 COMPUTE STATISTICS 
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "RTSP_DATA";
Trigger for MIN_KYC_WHITELIST_DETAILS
CREATE OR REPLACE EDITIONABLE TRIGGER "RTSP"."TRG_PROCESS_BATCH_DUPLICATES"\
AFTER INSERT ON RTSP.MIN_KYC_WHITELIST_DETAILS
BEGIN
  IF NOT trg_pkg.g_in_trigger THEN
    trg_pkg.g_in_trigger := TRUE;
    PROCESS_BATCH_DUPLICATES;
    trg_pkg.g_in_trigger := FALSE;
  END IF;
END;
/
ALTER TRIGGER "RTSP"."TRG_PROCESS_BATCH_DUPLICATES" ENABLE;

Procedure for MIN_KYC_WHITELIST_DETAILS
create or replace PROCEDURE PROCESS_BATCH_DUPLICATES AS
BEGIN
  -------------------------------------------------------------------
  -- Step 1: Mark first record per MOBILE and PROGRAM_NAME as valid
  -------------------------------------------------------------------
  MERGE INTO RTSP.MIN_KYC_WHITELIST_DETAILS tgt
  USING (
    SELECT ID, MOBILE,
           CASE
             WHEN ROW_NUMBER() OVER (PARTITION BY MOBILE, PROGRAM_NAME ORDER BY ID) = 1 THEN 'Y'
             ELSE 'N'
           END AS IS_VALID
    FROM RTSP.MIN_KYC_WHITELIST_DETAILS
  ) src
  ON (tgt.ID = src.ID)
  WHEN MATCHED THEN
    UPDATE SET tgt.IS_VALID = src.IS_VALID;
  -------------------------------------------------------------------
  -- Step 2a: Mark users as eligible if criteria met in CUSTOMER table AND IS_VALID = 'Y'
  -------------------------------------------------------------------
  MERGE INTO RTSP.MIN_KYC_WHITELIST_DETAILS mk
  USING (
    SELECT DISTINCT c.MOBILE
    FROM RTSP.CUSTOMER c
    JOIN RTSP.MIN_KYC_WHITELIST_DETAILS mk
      ON c.MOBILE = mk.MOBILE
    WHERE c.WALLET_ADDRESS IS NOT NULL
      AND c.AADHAR_ENABLED = 'Y'
      AND c.STATUS IN ('V', 'L', 'B')
      AND mk.IS_VALID = 'Y'
  ) cust
  ON (mk.MOBILE = cust.MOBILE)
  WHEN MATCHED THEN
    UPDATE SET
      mk.AADHAR_WALLET_AVAILABLE = 'Y',
      mk.ELIGIBLE_FOR_DISBURSEMENT = 'Y';
 
  -------------------------------------------------------------------
  -- Step 2b: Mark remaining users as not eligible (fallback)
  -------------------------------------------------------------------
  UPDATE RTSP.MIN_KYC_WHITELIST_DETAILS mk
  SET AADHAR_WALLET_AVAILABLE = 'N',
      ELIGIBLE_FOR_DISBURSEMENT = 'N'
  WHERE (mk.MOBILE, mk.IS_VALID) NOT IN (
    SELECT c.MOBILE, 'Y'
    FROM RTSP.CUSTOMER c
    JOIN RTSP.MIN_KYC_WHITELIST_DETAILS mk2
      ON c.MOBILE = mk2.MOBILE
    WHERE c.WALLET_ADDRESS IS NOT NULL
      AND c.AADHAR_ENABLED = 'Y'
      AND c.STATUS IN ('V', 'L', 'B')
      AND mk2.IS_VALID = 'Y'
  );
 
  -------------------------------------------------------------------
  -- Step 3: Update REMARKS based on CUSTOMER.STATUS with priority
  -------------------------------------------------------------------
  MERGE INTO RTSP.MIN_KYC_WHITELIST_DETAILS mk
  USING (
    SELECT MOBILE,
           CASE STATUS
             WHEN 'V' THEN 'Verified'
             WHEN 'L' THEN 'Locked'
             WHEN 'B' THEN 'Blocked'
             WHEN 'D' THEN 'Deregistered'
             ELSE 'Wallet Not Registered'
           END AS new_remarks
    FROM (
      SELECT MOBILE, STATUS,
             ROW_NUMBER() OVER (
               PARTITION BY MOBILE
               ORDER BY
                 CASE STATUS
                   WHEN 'V' THEN 1
                   WHEN 'L' THEN 2
                   WHEN 'B' THEN 3
                   WHEN 'D' THEN 4
                   ELSE 5
                 END
             ) AS rn
      FROM RTSP.CUSTOMER
    )
    WHERE rn = 1
  ) cust
  ON (mk.MOBILE = cust.MOBILE)
  WHEN MATCHED THEN
    UPDATE SET mk.REMARKS = cust.new_remarks;
 
EXCEPTION
  WHEN OTHERS THEN
    -- Optional: Log the error or raise an alert
    RAISE;
  COMMIT;
END PROCESS_BATCH_DUPLICATES;

Added Indexing for the Customer table:
 CREATE INDEX "RTSP"."CUSTOMER_IDX" ON "RTSP"."CUSTOMER" ("MOBILE", "STATUS", "AADHAR_ENABLED")
  PCTFREE 10 INITRANS 2 MAXTRANS 255 COMPUTE STATISTICS
  STORAGE(INITIAL 65536 NEXT 1048576 MINEXTENTS 1 MAXEXTENTS 2147483645
  PCTINCREASE 0 FREELISTS 1 FREELIST GROUPS 1
  BUFFER_POOL DEFAULT FLASH_CACHE DEFAULT CELL_FLASH_CACHE DEFAULT)
  TABLESPACE "RTSP_DATA";
 Customer:
We added two columns for customer:
ALTER TABLE RTSP.CUSTOMER
  ADD (
    AADHAR_ENABLED CHAR(1 BYTE) DEFAULT 'N',
    AADHAAR_USER_NAME VARCHAR2(255 BYTE) DEFAULT NULL
  );
Indexing for Customer:
create index rtsp.customer_idx  on rtsp.customer(mobile,status,aadhar_enabled);
PCBDC_DISBURSEMENT_LOG
ALTER TABLE RTSP.PCBDC_DISBURSEMENT_LOG
  ADD (
    AADHAR_LAST_FOUR_DIGITS VARCHAR2(20 BYTE),
    WHITELIST_BATCH_ID VARCHAR2(200 BYTE),
    PROGRAM_NAME VARCHAR2(256 BYTE)
  );
Indexing for PCBDC_DISBURSEMENT_LOG
create index mobile_idx on rtsp.PCBDC_DISBURSEMENT_LOG (mobile,PROGRAM,PCBDC_STATUS);
PCBDC_SPONSOR_RULE_LOG
RTSP.PCBDC_SPONSOR_RULE_LOG
  ADD (
    NEGATIVE_MCC_LIST VARCHAR2(100 BYTE) DEFAULT NULL,
    MCC_TYPE VARCHAR2(100 BYTE) DEFAULT NULL
  ); 
PCBDC_SPONSOR_RULE
RTSP.PCBDC_SPONSOR_RULE 
  ADD (
    NEGATIVE_MCC_LIST VARCHAR2(100 BYTE) DEFAULT NULL,
    MCC_TYPE VARCHAR2(100 BYTE) DEFAULT NULL
  );
Indexing for PCBDC_SPONSOR_RULE
CREATE INDEX "RTSP"."PCBDC_SPONSOR_RULE_IDX" ON "RTSP"."PCBDC_SPONSOR_RULE" ("RULE_ID", "AUTH_BY", "STATUS")
KYC_LIMITS: Added new recode for MIN KYC users
Insert into KYC_LIMITS (ID,TYPE,CAPACITY_LIMIT,PER_DAY_LOAD_LIMIT,PER_DAY_UNLOAD_LIMIT,PER_DAY_TFR_INWARD_LIMIT,PER_DAY_TFR_OUTWARD_LIMIT,TXN_LOAD_COUNT,TXN_TFR_INWARD_COUNT,TXN_UNLOAD_COUNT,TXN_TFR_OUTWARD_COUNT,PER_TXN_LIMIT,COOLING_LIMIT,LIMIT_TYPE,MONTHLY_TRF_OUTWARD_LIMIT,MONTHLY_TRF_OUTWARD_COUNT) values (158,'MIN',100000,5000,5000,50000,5000,50,50,50,50,5000,'N','MIN',500000,50);
RC Table:
Need to insert the some of the records in RC table
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0146','Aadhaar Timeout');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0147','Aadhaar incorrect datatype');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0148','Aadhaar incorrect reference');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0149','Aadhaar format incorrect');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0150','Aadhaar key blocked');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0151','Aadhaar data not available');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0152','Aadhaar duplicate');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0153','Aadhar invalid');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0154','Aadhaar not found');
Insert into RTSP.RC(MNEMONIC,DESCRIPTION) values ('cbs.request.failed.0155','Aadhaar technical error');
RC_LOCALE Table:
Need to insert the some of the records in RC_LOCALE table to take the id from the RC table while inserting the records for(Locale values(APP,SYS,PSO))
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1263,'SYS','0146','aadhaar Authentication Failed. Please try again later',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1267,'SYS','0147','Invalid aadhaar Number. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1268,'SYS','0148','Invalid Virtual Reference Number. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1269,'SYS','0149','Invalid aadhaar Number. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1270,'SYS','0150','Virtual Reference Number not found. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1271,'SYS','0151','aadhaar Number not found. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1272,'SYS','0152','Aadhaar Number already exists. Please input unique value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1273,'SYS','0153','Duplicate Virtual Reference Number. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1274,'SYS','0154','aadhaar Number not found. Please input correct value',null);
Insert into RTSP.RC_LOCALE (ID,LOCALE,RESULTCODE,RESULTINFO,EXTENDEDRESULTCODE) values (1275,'SYS','0155','Technical Error. Please try again late',null);





