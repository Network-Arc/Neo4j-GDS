# 🧠 Graph-Based Fraud Detection and Node Analysis with Neo4j

This technical notebook presents a complete graph-based approach for analyzing financial transaction networks using **Neo4j** and its **Graph Data Science (GDS)** library. The goal is to uncover hidden patterns, detect potentially fraudulent behaviors, and explore node-level relationships and influence in the graph.

---

## 📦 Part 1: Initial Graph Data Model

The following Cypher queries define the structure of the graph database, creating nodes for accounts and transactions, and relationships such as `TRANSFERRED` and `SENT`.
Part 1: Initial Graph Data Model
CREATE CONSTRAINT FOR (s:Store) REQUIRE s.sid IS UNIQUE;
CREATE CONSTRAINT FOR (c:Client) REQUIRE c.cid IS UNIQUE;
CREATE CONSTRAINT FOR (t:TFN) REQUIRE t.TFN IS UNIQUE;
CREATE CONSTRAINT FOR (p:Phone) REQUIRE p.Phone IS UNIQUE;
CREATE CONSTRAINT FOR (e:Email) REQUIRE e.email IS UNIQUE;
// Load clients.csv and create Client, Email, Phone, and TFN nodes, and relationships between them
LOAD CSV WITH HEADERS FROM 'file:///clients.csv' AS row
MERGE (c:Client {cid: toInteger(row.id)})
SET
c.name = row.name,
c.flagged = toBoolean(row.flagged)
MERGE (e:Email {email: row.email})
MERGE (c)-[:HAS_EMAIL]->(e)
MERGE (p:Phone {Phone: row.phone})
MERGE (c)-[:HAS_PHONE]->(p)
MERGE (t:TFN {TFN: row.tfn})
MERGE (c)-[:HAS_TFN]->(t);

// Load stores.csv and create properties Store ID and Store Name.
LOAD CSV WITH HEADERS
FROM 'file:///stores.csv' AS row
MERGE (s:Store {
sid: row.id,
name: row.name
});
// Load purchase.csv and create node purchase and store properties of the purchase such as amount, time, From address and To address.
LOAD CSV WITH HEADERS FROM 'file:///purchase.csv' AS row
MERGE (p:Purchase {
amount: toFloat(row.amount),
time: datetime("2024-05-12T00:00:00") + duration({seconds: toInteger(row.timeOffset)}), // Convert timeOffset into a timestamp
idTo: row.idTo,
idFrom: toInteger(row.idFrom),
nameTo: row.nameTo,
nameFrom: row.nameFrom
});
// Load purchase.csv and create node Transaction node and store properties of the transaction such as amount, time, From address, To address and type of transaction that is either “Purchase” or “Transfer”.
LOAD CSV WITH HEADERS FROM 'file:///purchase.csv' AS row
MERGE (tx:Transaction {
amount: toFloat(row.amount),
time: datetime("2024-05-12T00:00:00") + duration({seconds: toInteger(row.timeOffset)}), // Convert timeOffset into a timestamp
idTo: row.idTo,
idFrom: toInteger(row.idFrom),
nameTo: row.nameTo,
nameFrom: row.nameFrom,
type:"Purchase"
});
// Create relationship between the client and store nodes which performed the purchase relationship and sent to relationship using purchase nodes
MATCH (c:Client), (p:Purchase)
WHERE c.cid = p.idFrom
MERGE (c)-[:PERFORMED]->(p);
MATCH (s:Store), (p:Purchase)
WHERE s.sid = p.idTo
MERGE (p)-[:TO]->(s);

// Create relationship between the client and store nodes which performed the transaction relationship and sent to relationship using Transaction nodes
MATCH (c:Client), (tx:Transaction)
WHERE c.cid = tx.idFrom
MERGE (c)-[:PERFORMED]->(tx);
MATCH (s:Store), (tx:Transaction)
WHERE s.sid = tx.idTo
MERGE (tx)-[:TO]->(s);
// Load purchase.csv and create node purchase and store properties of the transfer such as amount, time, From address and To address.
LOAD CSV WITH HEADERS FROM 'file:///xfer.csv' AS row
MERGE (t:Transfer {
amount: toFloat(row.amount),
time: datetime("2024-05-12T00:00:00") + duration({seconds: toInteger(row.timeOffset)}), // Convert timeOffset to datetime
idTo: toInteger(row.idTo),
idFrom: toInteger(row.idFrom),
nameTo: row.nameTo,
nameFrom: row.nameFrom
});
// Create relationship between the client and store nodes which performed the transfer relationship and sent to relationship using Transaction nodes.
LOAD CSV WITH HEADERS FROM 'file:///xfer.csv' AS row
MERGE (tx:Transaction {
amount: toFloat(row.amount),
time: datetime("2024-05-12T00:00:00") + duration({seconds: toInteger(row.timeOffset)}), // Convert timeOffset to datetime
idTo: toInteger(row.idTo),
idFrom: toInteger(row.idFrom),
nameTo: row.nameTo,
nameFrom: row.nameFrom,
type:"Transfer"
});
// Create relationship between the client and store nodes which performed the transaction relationship and sent to relationship using Transaction nodes
MATCH (sender:Client), (tx:Transaction)
WHERE sender.cid = tx.idFrom
MERGE (sender)-[:PERFORMED]->(tx);
MATCH (receiver:Client), (tx:Transaction)
WHERE receiver.cid = tx.idTo
MERGE (tx)-[:TO]->(receiver);
// Create relationship between the client and store nodes which performed the transfer relationship and sent to relationship using transfer nodes
MATCH (sender:Client), (t:Transfer)
WHERE sender.cid = t.idFrom
MERGE (sender)-[:PERFORMED]->(t);
MATCH (receiver:Client), (t:Transfer)
WHERE receiver.cid = t.idTo
MERGE (t)-[:TO]->(receiver);

```

---

## 🔍 Part 2: Analytical Querying for Pattern Detection

This section includes Cypher queries designed to explore and extract patterns from the graph—such as high-value transactions, clusters of activity, and direct/indirect connections between accounts.

```cypher

 
Part 2: Initial Queries

Problem 1)
// Set up the start and end times for the query
WITH datetime('2024-05-12T10:00:00') AS startTime, datetime('2024-05-12T14:00:00') AS endTime
// Match clients who made purchases in that time range
MATCH (c:Client)-[:PERFORMED]->(tx:Purchase)
WHERE tx.time >= startTime AND tx.time <= endTime
// Calculate the total spending for each client
WITH c.name AS clientName, SUM(tx.amount) AS totalSpent
// Return the client who spent the most
RETURN clientName, totalSpent
ORDER BY totalSpent DESC
LIMIT 1;
OUTPUT:
Problem 2)

MATCH (c:Client)
// Match all `Client` nodes.
OPTIONAL MATCH (c)-[:PERFORMED]->(tx:Transaction)
// Optionally match the outgoing `Transaction` relationships where the `Client` performed transactions (either a Purchase or Transfer).
WHERE tx.type IN ["Purchase", "Transfer"]
// Filter the outgoing transactions to include only those of type "Purchase" or "Transfer".
WITH c, SUM(tx.amount) AS outgoing
// Aggregate the total amount of outgoing transactions per client and store it as `outgoing`.
OPTIONAL MATCH (txIn:Transaction)-[:TO]->(c)
// Optionally match the incoming transactions where the `Client` was the receiver of a "Transfer" transaction.
WHERE txIn.type = "Transfer"
// Filter the incoming transactions to include only those of type "Transfer".
WITH c.name AS name, outgoing, SUM(txIn.amount) AS incoming
// Aggregate the total amount of incoming transfers for each client and store it as `incoming`.
WITH name, incoming, outgoing, incoming - outgoing AS balance
// Calculate the `balance` for each client by subtracting the outgoing from the incoming amount.
WHERE balance < 0
// Filter to show only clients with a negative balance (i.e., they spent more than they received).
RETURN name, balance, outgoing AS big_spend
// Return the name of the client, their negative balance, and the total outgoing amount (big_spend).
ORDER BY balance ASC
// Sort the results by the most negative balance first (those who spent the most relative to their incoming transactions).
LIMIT 5;
// Limit the result to the top 5 clients with the highest overspending (negative balance).
OUTPUT:

Problem 3)
// Match clients who performed a transfer followed by a purchase at the store "Woods"
MATCH (sender:Client)-[:PERFORMED]->(transfer:Transaction {type: "Transfer"})-[:TO]->(receiver:Client),
(receiver)-[:PERFORMED]->(purchase:Transaction {type: "Purchase"})-[:TO]->(s:Store {name: 'Woods'})

// Ensure that the transfer happened before the purchase
WHERE transfer.time < purchase.time
// Calculate total amount received by the receiver and total amount spent at the store
WITH receiver, sum(transfer.amount) as total_received, sum(purchase.amount) as total_spent
// Filter for cases where spending is at least 5% of the received amount
WHERE total_spent >= 0.05 * total_received
// Return the receiver's name, percentage spent, total transfer amount, and total spent
RETURN 
receiver.name as name,
(total_spent / total_received * 100) as percentage,
total_received as total_xfer,
total_spent as total_purchase

// Order by the percentage of funds spent
ORDER BY percentage DESC

OUTPUT:

Problem 4)

// Order transactions for each client and create FIRST, NEXT, and LAST relationships
MATCH (c:Client)-[:PERFORMED]->(t:Transaction)
WITH c, t
ORDER BY c.cid, t.time
WITH c, collect(t) as orderedTransactions
WITH c, orderedTransactions,
,

// Identify the first and last transaction for each client
orderedTransactions[0] as firstTx,
orderedTransactions[-1] as lastTx
MERGE (c)-[:FIRST_TX]->(firstTx)
MERGE (c)-[:LAST_TX]->(lastTx)
// Create NEXT relationships between consecutive transactions
WITH c, orderedTransactions
UNWIND range(0, size(orderedTransactions)-2) as i
WITH c, orderedTransactions[i] as t1, orderedTransactions[i+1] as t2
MERGE (t1)-[:NEXT]->(t2)


OUTPUT:
Below represent one of the client node “Carson Murray” having relationship  consisting of next, first and last transaction in chain

```

---

## 🧠 Part 3: Graph Data Science (GDS) Workflow

Advanced analytics using Neo4j GDS for:
- Centrality detection
- Community detection
- Node similarity and anomaly identification

```cypher

 
Part 3: Graph Data Science

Part A:
i)
// Step 1: Create a projection of the graph for 'client-identifiers'
// This step creates a graph in the GDS (Graph Data Science) catalog, projecting nodes of 'Client', 'Email', 'Phone', and 'TFN' labels.
// Relationships of types HAS_EMAIL, HAS_PHONE, and HAS_TFN are set as undirected, meaning they can be traversed in both directions.
CALL gds.graph.project(
  'client-identifiers', // The name of the graph
  ['Client', 'Email', 'Phone', 'TFN'], // Nodes to be included in the projection
  {
    HAS_EMAIL: {orientation: 'UNDIRECTED'}, // Relationship type for email associations
    HAS_PHONE: {orientation: 'UNDIRECTED'}, // Relationship type for phone associations
    HAS_TFN: {orientation: 'UNDIRECTED'} // Relationship type for tax file number (TFN) associations
  }
);

// Step 2: Run the Louvain community detection algorithm to find communities
// The Louvain algorithm identifies clusters of nodes based on their connections, assigning a community ID to each node.
CALL gds.louvain.stream('client-identifiers')
YIELD nodeId, communityId
// Match the Client nodes using the nodeId from the GDS result to assign them to a community.
MATCH (c:Client)
WHERE id(c) = nodeId AND c.name IS NOT NULL
// Return the name of the client and the community they belong to.
RETURN c.name AS clientName, communityId
ORDER BY communityId;

ii)
// Step 3: Filter communities with 5 or more members and assign them a groupId
// This step groups nodes that belong to larger communities (with at least 5 members).
CALL gds.louvain.stream('client-identifiers')
YIELD nodeId, communityId
WITH communityId, collect(nodeId) AS members
// Filter out communities that have fewer than 5 members.
WHERE size(members) >= 5
WITH communityId, members
UNWIND members AS memberId
// For each member of the community, match the Client node and assign a groupId equal to the communityId.
MATCH (c:Client) WHERE id(c) = memberId AND c.name IS NOT NULL
SET c.groupId = communityId
// Return the communityId and the size of the group.
RETURN communityId, count(c) AS groupSize
ORDER BY groupSize DESC;

iii)
// Step 4: Find the largest group and retrieve the relationships for its members
// This query finds the largest group by size and retrieves its members' email, phone, or TFN relationships.
MATCH (c:Client)
WHERE c.groupId IS NOT NULL
WITH c.groupId AS groupId, count(c) AS groupSize
// Limit to the largest group by size.
ORDER BY groupSize DESC
LIMIT 1
// Match relationships for members of this group (email, phone, or TFN).
MATCH (c:Client {groupId: groupId})-[r:HAS_EMAIL|HAS_PHONE|HAS_TFN]-(i)
// Return the Client node, the relationship, and the associated identity (email, phone, or TFN).
RETURN c, r, i;

OUTPUT:



Part B:
i) 
// Match clients that are part of a fraud group
MATCH (c:Client)
// Filter out clients who are not assigned to a group
WHERE c.groupId IS NOT NULL
// Group clients by their groupId and count the number of members in each group
WITH c.groupId AS groupId, COUNT(c) AS groupSize
// Only consider groups with more than 5 members
WHERE groupSize > 5
// Return the groupId and the corresponding size, ordering by the largest groups
RETURN groupId, groupSize
ORDER BY groupSize DESC

OUTPUT:


ii)
// Match transactions performed by clients within large fraud groups (e.g., groupId in the provided list)
// to clients outside of their immediate group
MATCH (c1:Client)-[:PERFORMED]->(t:Transaction)-[:TO]->(c2:Client)
// Ensure that the second client is either not part of a group or belongs to a different group
WHERE c1.groupId IN [4237, 4252, 4239, 4248, 4352, 4343, 4281, 4265, 4313, 4304, 4308, 4300, 4309]
AND (c2.groupId IS NULL OR c2.groupId <> c1.groupId)
// Return details of the transaction: client IDs, group IDs, transaction amount, and time
RETURN c1.cid AS fromClient, c1.groupId AS fromGroup, 
       c2.cid AS toClient, c2.groupId AS toGroup, 
       t.amount AS amount, t.time AS time
// Order the results by transaction amount, descending
ORDER BY t.amount DESC
LIMIT 10

OUTPUT:

// Create a graph projection in GDS using the 'fraud-graph' name.
// Include Client and Transaction nodes and undirected relationships between them.
CALL gds.graph.project(
  'fraud-graph',
  ['Client', 'Transaction'],  // Specify the relevant node labels to be included
  {
    PERFORMED: {orientation: 'UNDIRECTED'},  // Use undirected relationships between clients and transactions
    TO: {orientation: 'UNDIRECTED'}  // Ensure all necessary relationships are included
  }
)
iii)
// Run PageRank on the 'fraud-graph' to identify central nodes and write the results to a 'pagerank' property
CALL gds.pageRank.write('fraud-graph', {
  writeProperty: 'pagerank'
})
// Return the number of node properties updated by the algorithm
YIELD nodePropertiesWritten;

// Match transactions performed by clients from group 4237 to clients outside their group
MATCH (sender:Client {groupId: 4237})-[:PERFORMED]->(tx:Transaction)-[:TO]->(receiver:Client)
// Filter to only include transactions where the receiver is in a different group
WHERE receiver.groupId IS NULL OR receiver.groupId <> sender.groupId
// Order the results by sender's PageRank to identify central actors
WITH sender, receiver, tx
ORDER BY sender.pagerank DESC  // Sort by PageRank to find the most central actors
// Return the sender's and receiver's names, group information, transaction details, and PageRank
RETURN sender.name AS senderName, sender.groupId AS senderGroupId, sender.pagerank AS senderPageRank, 
       receiver.name AS receiverName, receiver.groupId AS receiverGroupId, 
       tx.amount AS transactionAmount, tx.time AS transactionTime
// Sort the results by PageRank and transaction amount in descending order
ORDER BY sender.pagerank DESC, tx.amount DESC;


iv)

YouTube Video Link: https://youtu.be/aa6J7ABUfQ8
```

---

### ✅ Outcome

By combining structured graph modeling with analytical queries and GDS algorithms, this approach enables efficient detection of suspicious activity in complex financial networks. It can be extended to real-world scenarios such as:
- Anti-money laundering (AML)
- Insider threat detection
- Supply chain risk monitoring

