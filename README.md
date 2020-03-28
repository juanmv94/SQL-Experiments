# SQL-Experiments

Here you can find some complex SQL querys for solving problems or just do interesting outputs. They are implemented in SQLite but they could be ported to other SQL platforms like Oracle.

**Table of Contents**

[TOCM]

[TOC]

## Commond Child (Dynamic algorithm)
This is a solution to the [*Common Child* problem from HackerRank](https://www.hackerrank.com/challenges/common-child/ "Common Child problem from HackerRank") also know as the [*Longest common subsequence* problem](https://en.wikipedia.org/wiki/Longest_common_subsequence_problem "Longest common subsequence problem").

It should return the longest string which is a common child of the input strings. For example, ABCD and ABDC have two children with maximum length 3, ABC and ABD. They can be formed by eliminating either the D or C from both strings. Note that we will not consider ABCD as a common child because we can't rearrange characters.

My dynamic algorithm for solving this problem in C is the following:

```c
int commonChild(char* s1, char* s2) {
    int x=strlen(s1),y=strlen(s2);
    //memory allocation
    int** mat=malloc((x+1)*sizeof(int*));
    for (int i=0;i<x;i++) {
        mat[i]=malloc((y+1)*sizeof(int));
        mat[i][y]=0;
    }
    mat[x]=calloc((y+1), sizeof(int));
    //dinamic algorithm
    for (int i=(x-1);i>=0;i--) for (int j=(y-1);j>=0;j--) {
        mat[i][j] = (s1[i]==s2[j]) ? (1+mat[i+1][j+1]) : MAX(mat[i+1][j],mat[i][j+1]);
    }
    int result=mat[0][0];
    for (int i=0;i<x;i++) free(mat[i]);
    free(mat);
    return result;
}
```

As you see, this algorithm is dynamic since it is solved using results from subproblems (substrings of the input strings).  We generate a matrix with the results of all substring combinations (length string1 x length string2)
* When one of the input strings is empty, the result is **0**.
* When the last character of both strings strings are equal, the result is **1+matrix(x-1,y-1)** (both strings without the last character)
* Else, the result is the maximum between both strings with one of them missing the last character, that is discarded, beeing **max ( matrix[x-1][y], matrix[x][y-1] )**.

Example:

|  | **[]** | **A** | **AB** |  **ABC** | **ABCD** | **ABCDE** |  **ABCDEF** |
| ------------ |------------ | ------------ | ------------ | ------------ | ------------ | ------------ | ------------ |
| **[]** | 0 | 0 | 0 | 0 | 0 | 0 | 0 |
| **F** | 0 | 0 | 0 | 0 | 0 | 0 | 1 |
| **FB** | 0 | 0 | 1 | 1 | 1 | 1 | 1 |
| **FBD** | 0 | 0 | 1 | 1 | 2 | 2 | 2 |
| **FBDA** | 0 | 1 | 1 | 1 | 2 | 2| 2 |
| **FBDAM** | 0 | 1 | 1 | 1 | 2 | 2| 2 |
| **FBDAMN** | 0 | 1 | 1 | 1 | 2 | 2| 2 |

But... how to implement all this stuff in a SQL query?
* We can use a recursive query to iterate all the matrix indexes in a order. We use CASE statements to check if we are at the last columnindex from the matrix, and the increment then rowindex, setting the columnindex to 0. The recursive query will stop with X=LENGTH(string1) AND Y=LENGTH(string2)
* since subqueries inside a recursive queries are not allowed... What can we do to check the previous generated results? We are going to generate a field named A with an array of the previous results. They are going to be 0-padded 4-character numbers, so we can get a previous result with the SUBSTR() function.

The final query is the following:

```sql
WITH
INPUTS(I1,I2) AS (
	SELECT 'HARRY','SALLY'
	UNION ALL SELECT 'AA','BB'
	UNION ALL SELECT 'SHINCHAN','NOHARAAA'
	UNION ALL SELECT 'ABCDEF','FBDAMN'
),
RESULTS(I1,I2,X,Y,A) AS (
	SELECT I1,I2,0,1,'0000'
	FROM INPUTS UNION ALL
	SELECT I1,I2,
		CASE WHEN X=LENGTH(I1) THEN 0 ELSE X+1 END,
		CASE WHEN X=LENGTH(I1) THEN Y+1 ELSE Y END,
		CASE WHEN X=LENGTH(I1) THEN '0000' ELSE
		SUBSTR('0000'||(
			CASE WHEN SUBSTR(I1,(CASE WHEN X=LENGTH(I1) THEN 1 ELSE X+1 END),1) = SUBSTR(I2,(CASE WHEN X=LENGTH(I1)THEN Y+1 ELSE Y END),1)
			THEN 1 + CAST(SUBSTR(A,1+4*(LENGTH(I1)+1),4) AS INTEGER)
			ELSE MAX(CAST(SUBSTR(A,1+4*LENGTH(I1),4) AS INTEGER),CAST(SUBSTR(A,1,4) AS INTEGER)) END
		),-4,4) END || A
	FROM RESULTS
	WHERE NOT (X=LENGTH(I1) AND Y=LENGTH(I2))
),
FRESULTS(I1,I2,R) AS (
	SELECT I1,I2,CAST(SUBSTR(A,1,4) AS INTEGER) FROM RESULTS WHERE X=LENGTH(I1) AND Y=LENGTH(I2)
)
SELECT * FROM FRESULTS;
```