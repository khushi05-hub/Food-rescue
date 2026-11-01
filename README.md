
-- =====================================================
-- FOODRESCUE
-- SMART FOOD WASTE AND DONATION MANAGEMENT SYSTEM
-- ORACLE SQL + PL/SQL MINI PROJECT
-- =====================================================

SET SERVEROUTPUT ON;

-- =====================================================
-- PART 1: CREATE DATABASE TABLES
-- =====================================================

-- 1. CATEGORIES TABLE

CREATE TABLE Categories (
    category_id NUMBER PRIMARY KEY,
    category_name VARCHAR2(50) NOT NULL
);

-- 2. DONORS TABLE

CREATE TABLE Donors (
    donor_id NUMBER PRIMARY KEY,
    donor_name VARCHAR2(100) NOT NULL,
    contact_number VARCHAR2(15),
    address VARCHAR2(150)
);

-- 3. NGOS TABLE

CREATE TABLE NGOs (
    ngo_id NUMBER PRIMARY KEY,
    ngo_name VARCHAR2(100) NOT NULL,
    contact_number VARCHAR2(15),
    address VARCHAR2(150)
);

-- 4. FOOD ITEMS TABLE

CREATE TABLE Food_Items (
    food_id NUMBER PRIMARY KEY,
    food_name VARCHAR2(100) NOT NULL,
    category_id NUMBER,
    donor_id NUMBER,
    quantity_kg NUMBER(10,2) CHECK (quantity_kg >= 0),
    expiry_date DATE,
    food_status VARCHAR2(20) DEFAULT 'Available',
    FOREIGN KEY (category_id) REFERENCES Categories(category_id),
    FOREIGN KEY (donor_id) REFERENCES Donors(donor_id)
);

-- 5. REQUESTS TABLE

CREATE TABLE Requests (
    request_id NUMBER PRIMARY KEY,
    ngo_id NUMBER,
    food_id NUMBER,
    requested_quantity NUMBER(10,2),
    request_date DATE DEFAULT SYSDATE,
    request_status VARCHAR2(20) DEFAULT 'Pending',
    FOREIGN KEY (ngo_id) REFERENCES NGOs(ngo_id),
    FOREIGN KEY (food_id) REFERENCES Food_Items(food_id),
    CHECK (requested_quantity > 0)
);

-- 6. DISTRIBUTIONS TABLE

CREATE TABLE Distributions (
    distribution_id NUMBER PRIMARY KEY,
    request_id NUMBER UNIQUE,
    distributed_quantity NUMBER(10,2),
    distribution_date DATE DEFAULT SYSDATE,
    FOREIGN KEY (request_id) REFERENCES Requests(request_id),
    CHECK (distributed_quantity > 0)
);

-- 7. AUDIT LOG TABLE

CREATE TABLE Audit_Log (
    audit_id NUMBER PRIMARY KEY,
    distribution_id NUMBER,
    action_details VARCHAR2(200),
    log_date DATE DEFAULT SYSDATE
);

-- =====================================================
-- PART 2: CREATE SEQUENCES
-- =====================================================

CREATE SEQUENCE distribution_seq START WITH 1 INCREMENT BY 1;
CREATE SEQUENCE audit_seq START WITH 1 INCREMENT BY 1;

-- =====================================================
-- PART 3: INSERT SAMPLE DATA
-- =====================================================

INSERT INTO Categories VALUES (1, 'Cooked Food');
INSERT INTO Categories VALUES (2, 'Fruits');
INSERT INTO Categories VALUES (3, 'Vegetables');
INSERT INTO Categories VALUES (4, 'Bakery');
INSERT INTO Categories VALUES (5, 'Packaged Food');

INSERT INTO Donors VALUES
(1, 'Green Leaf Restaurant', '9876543210', 'Chandigarh');

INSERT INTO Donors VALUES
(2, 'Royal Hotel', '9876543211', 'Mohali');

INSERT INTO Donors VALUES
(3, 'Fresh Bakery', '9876543212', 'Zirakpur');

INSERT INTO NGOs VALUES
(1, 'Helping Hands NGO', '9988776655', 'Chandigarh');

INSERT INTO NGOs VALUES
(2, 'Food For All', '9988776656', 'Mohali');

INSERT INTO NGOs VALUES
(3, 'Hope Foundation', '9988776657', 'Panchkula');

INSERT INTO Food_Items VALUES
(1, 'Vegetable Rice', 1, 1, 25, SYSDATE + 1, 'Available');

INSERT INTO Food_Items VALUES
(2, 'Fresh Apples', 2, 2, 20, SYSDATE + 3, 'Available');

INSERT INTO Food_Items VALUES
(3, 'Mixed Vegetables', 3, 1, 40, SYSDATE + 2, 'Available');

INSERT INTO Food_Items VALUES
(4, 'Fresh Bread', 4, 3, 30, SYSDATE + 2, 'Available');

INSERT INTO Food_Items VALUES
(5, 'Packaged Meals', 5, 2, 15, SYSDATE + 5, 'Available');

INSERT INTO Requests VALUES
(1, 1, 1, 10, SYSDATE, 'Pending');

INSERT INTO Requests VALUES
(2, 2, 2, 5, SYSDATE, 'Pending');

INSERT INTO Requests VALUES
(3, 3, 3, 15, SYSDATE, 'Pending');

COMMIT;

-- =====================================================
-- PART 4: CREATE AVAILABLE FOOD VIEW
-- =====================================================

CREATE OR REPLACE VIEW Available_Food AS
SELECT
    f.food_id,
    f.food_name,
    c.category_name,
    d.donor_name,
    f.quantity_kg,
    f.expiry_date
FROM Food_Items f
JOIN Categories c
    ON f.category_id = c.category_id
JOIN Donors d
    ON f.donor_id = d.donor_id
WHERE f.quantity_kg > 0
  AND f.expiry_date > SYSDATE
  AND f.food_status = 'Available';

-- =====================================================
-- PART 5: CREATE DONATION SUMMARY VIEW
-- =====================================================

CREATE OR REPLACE VIEW Donation_Summary AS
SELECT
    d.donor_name,
    COUNT(f.food_id) AS total_donations,
    NVL(SUM(f.quantity_kg), 0) AS total_quantity_kg
FROM Donors d
LEFT JOIN Food_Items f
    ON d.donor_id = f.donor_id
GROUP BY d.donor_name;

-- =====================================================
-- PART 6: APPROVE NGO REQUEST PROCEDURE
-- =====================================================

CREATE OR REPLACE PROCEDURE Approve_Request (
    p_request_id IN NUMBER
)
AS
    v_food_id NUMBER;
    v_requested NUMBER;
    v_available NUMBER;
    v_expiry DATE;
    v_status VARCHAR2(20);
BEGIN
    SELECT food_id, requested_quantity, request_status
    INTO v_food_id, v_requested, v_status
    FROM Requests
    WHERE request_id = p_request_id
    FOR UPDATE;

    IF v_status <> 'Pending' THEN
        RAISE_APPLICATION_ERROR(
            -20001, 'Request is not pending'
        );
    END IF;

    SELECT quantity_kg, expiry_date
    INTO v_available, v_expiry
    FROM Food_Items
    WHERE food_id = v_food_id;

    IF v_expiry <= SYSDATE THEN
        RAISE_APPLICATION_ERROR(
            -20002, 'Food has expired'
        );
    END IF;

    IF v_available < v_requested THEN
        RAISE_APPLICATION_ERROR(
            -20003, 'Insufficient food quantity'
        );
    END IF;

    UPDATE Requests
    SET request_status = 'Approved'
    WHERE request_id = p_request_id;

    COMMIT;

    DBMS_OUTPUT.PUT_LINE('Request approved successfully!');
    DBMS_OUTPUT.PUT_LINE('Request ID: ' || p_request_id);

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        ROLLBACK;
        RAISE_APPLICATION_ERROR(
            -20004, 'Request or food item not found'
        );
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/

-- =====================================================
-- PART 7: VALIDATE DISTRIBUTION TRIGGER
-- =====================================================

CREATE OR REPLACE TRIGGER Validate_Distribution
BEFORE INSERT ON Distributions
FOR EACH ROW
DECLARE
    v_requested NUMBER;
    v_available NUMBER;
    v_expiry DATE;
    v_status VARCHAR2(20);
    v_food_id NUMBER;
BEGIN
    SELECT r.requested_quantity,
           r.request_status,
           r.food_id
    INTO v_requested, v_status, v_food_id
    FROM Requests r
    WHERE r.request_id = :NEW.request_id;

    IF v_status <> 'Approved' THEN
        RAISE_APPLICATION_ERROR(
            -20005, 'Request must be approved first'
        );
    END IF;

    IF :NEW.distributed_quantity > v_requested THEN
        RAISE_APPLICATION_ERROR(
            -20006, 'Distribution exceeds requested quantity'
        );
    END IF;

    SELECT quantity_kg, expiry_date
    INTO v_available, v_expiry
    FROM Food_Items
    WHERE food_id = v_food_id
    FOR UPDATE;

    IF v_expiry <= SYSDATE THEN
        RAISE_APPLICATION_ERROR(
            -20007, 'Cannot distribute expired food'
        );
    END IF;

    IF :NEW.distributed_quantity > v_available THEN
        RAISE_APPLICATION_ERROR(
            -20008, 'Insufficient available stock'
        );
    END IF;
END;
/

-- =====================================================
-- PART 8: UPDATE STOCK AFTER DISTRIBUTION TRIGGER
-- =====================================================

CREATE OR REPLACE TRIGGER Update_Food_After_Distribution
AFTER INSERT ON Distributions
FOR EACH ROW
DECLARE
    v_food_id NUMBER;
BEGIN
    SELECT food_id
    INTO v_food_id
    FROM Requests
    WHERE request_id = :NEW.request_id;

    UPDATE Food_Items
    SET quantity_kg = quantity_kg - :NEW.distributed_quantity,
        food_status =
            CASE
                WHEN quantity_kg - :NEW.distributed_quantity = 0
                THEN 'Distributed'
                ELSE 'Available'
            END
    WHERE food_id = v_food_id;

    UPDATE Requests
    SET request_status = 'Completed'
    WHERE request_id = :NEW.request_id;

END;
/

-- =====================================================
-- PART 9: DISTRIBUTE FOOD PROCEDURE
-- =====================================================

CREATE OR REPLACE PROCEDURE Distribute_Food (
    p_request_id IN NUMBER,
    p_quantity IN NUMBER
)
AS
BEGIN
    IF p_quantity <= 0 THEN
        RAISE_APPLICATION_ERROR(
            -20009, 'Invalid distribution quantity'
        );
    END IF;

    INSERT INTO Distributions (
        distribution_id,
        request_id,
        distributed_quantity,
        distribution_date
    )
    VALUES (
        distribution_seq.NEXTVAL,
        p_request_id,
        p_quantity,
        SYSDATE
    );

    COMMIT;

    DBMS_OUTPUT.PUT_LINE('Food distributed successfully!');
    DBMS_OUTPUT.PUT_LINE('Request ID: ' || p_request_id);
    DBMS_OUTPUT.PUT_LINE('Quantity: ' || p_quantity || ' kg');

EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
/

-- =====================================================
-- PART 10: FUNCTION TO CHECK AVAILABLE FOOD
-- =====================================================

CREATE OR REPLACE FUNCTION Check_Food_Stock (
    p_food_id IN NUMBER
)
RETURN NUMBER
AS
    v_quantity NUMBER;
BEGIN
    SELECT quantity_kg
    INTO v_quantity
    FROM Food_Items
    WHERE food_id = p_food_id;

    RETURN v_quantity;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(
            -20010, 'Food item not found'
        );
END;
/

-- =====================================================
-- PART 11: AUDIT LOG TRIGGER
-- =====================================================

CREATE OR REPLACE TRIGGER Distribution_Audit
AFTER INSERT ON Distributions
FOR EACH ROW
BEGIN
    INSERT INTO Audit_Log (
        audit_id,
        distribution_id,
        action_details,
        log_date
    )
    VALUES (
        audit_seq.NEXTVAL,
        :NEW.distribution_id,
        'Food distributed: ' ||
        :NEW.distributed_quantity || ' kg',
        SYSDATE
    );
END;
/

-- =====================================================
-- PART 12: PENDING REQUESTS CURSOR PROCEDURE
-- =====================================================

CREATE OR REPLACE PROCEDURE Show_Pending_Requests
AS
    CURSOR request_cursor IS
        SELECT
            r.request_id,
            n.ngo_name,
            f.food_name,
            r.requested_quantity,
            r.request_status
        FROM Requests r
        JOIN NGOs n
            ON r.ngo_id = n.ngo_id
        JOIN Food_Items f
            ON r.food_id = f.food_id
        WHERE r.request_status = 'Pending';

BEGIN
    DBMS_OUTPUT.PUT_LINE('===== PENDING NGO REQUESTS =====');

    FOR rec IN request_cursor
    LOOP
        DBMS_OUTPUT.PUT_LINE(
            'Request ID: ' || rec.request_id ||
            ' | NGO: ' || rec.ngo_name ||
            ' | Food: ' || rec.food_name ||
            ' | Quantity: ' || rec.requested_quantity ||
            ' kg'
        );
    END LOOP;
END;
/

-- =====================================================
-- PART 13: FOOD EXPIRY REPORT PROCEDURE
-- =====================================================

CREATE OR REPLACE PROCEDURE Expiring_Food_Report
AS
    CURSOR food_cursor IS
        SELECT food_name, quantity_kg, expiry_date
        FROM Food_Items
        WHERE expiry_date BETWEEN SYSDATE
        AND SYSDATE + 1
        AND quantity_kg > 0
        AND food_status = 'Available';

BEGIN
    DBMS_OUTPUT.PUT_LINE('===== FOOD EXPIRING WITHIN 24 HOURS =====');

    FOR rec IN food_cursor
    LOOP
        DBMS_OUTPUT.PUT_LINE(
            'Food: ' || rec.food_name ||
            ' | Quantity: ' || rec.quantity_kg ||
            ' kg | Expiry: ' || rec.expiry_date
        );
    END LOOP;
END;
/

-- =====================================================
-- PART 14: DEMONSTRATION
-- =====================================================

BEGIN
    DBMS_OUTPUT.PUT_LINE('====================================');
    DBMS_OUTPUT.PUT_LINE('       FOODRESCUE SYSTEM');
    DBMS_OUTPUT.PUT_LINE('====================================');

    -- Display pending requests
    Show_Pending_Requests;

    -- Approve request 1
    Approve_Request(1);

    -- Distribute 10 kg of Vegetable Rice
    Distribute_Food(1, 10);

    -- Check remaining stock
    DBMS_OUTPUT.PUT_LINE(
        'Remaining Vegetable Rice: ' ||
        Check_Food_Stock(1) || ' kg'
    );

    -- Display expiry report
    Expiring_Food_Report;

    DBMS_OUTPUT.PUT_LINE('====================================');
    DBMS_OUTPUT.PUT_LINE('       FOODRESCUE COMPLETED');
    DBMS_OUTPUT.PUT_LINE('====================================');
END;
/

-- =====================================================
-- PART 15: FINAL REPORT QUERIES
-- =====================================================

-- Available food

SELECT * FROM Available_Food;

-- Donor contribution summary

SELECT * FROM Donation_Summary
ORDER BY total_quantity_kg DESC;

-- NGO request details

SELECT
    r.request_id,
    n.ngo_name,
    f.food_name,
    r.requested_quantity,
    r.request_status
FROM Requests r
JOIN NGOs n ON r.ngo_id = n.ngo_id
JOIN Food_Items f ON r.food_id = f.food_id;

-- Distribution details

SELECT * FROM Distributions;

-- Audit records

SELECT * FROM Audit_Log;

-- Total food distributed

SELECT
    NVL(SUM(distributed_quantity), 0)
    AS total_food_distributed_kg
FROM Distributions;
