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

## Next Gen Coder Challenge \#38
[Coding challenge 38 from NextGenCoder](https://www.instagram.com/p/B-SgN6jAtkt/ "NextGenCoder"):

*Write a program which will take one list containing bags of candies, each bag may contain 1, 2, or 3 candies ex. [2, 3, 1, 3] now give "Yes" as output if we can split them into 3 else give "No" as output*

We should split the list into 3 lists with the same sum of candies.

With inputs:
* [3,2,2,2]
* [3,3,3,2,2,2,2,1]
* [2,3,1,3]
* [3,2,3,3]

I get outputs
* [3,2,2,2] No
* [3,3,3,2,2,2,2,1] Yes   3\*2 , 3+2+1 , 2\*3
* [2,3,1,3] Yes   3 , 3 , 2+1
* [3,2,3,3] No

As you can see, in my SQL implementation I aditionally explain the possible solutions (This adds some additional optional lines to my query).

Here I use a simple exhaustive (but filtered) search for finding the solutions.

```sql
WITH INPUTS(G) AS (		--TABLE WITH INPUTS
	SELECT '[3,2,2,2]'
	UNION ALL SELECT '[3,3,3,2,2,2,2,1]'
	UNION ALL SELECT '[2,3,1,3]'
	UNION ALL SELECT '[3,2,3,3]')
,PINPUTS(G,O3,O2,O1) AS (	--INPUTS AND COUNT OF 3s, 2s, AND 1s
	SELECT G,
		LENGTH(G)-LENGTH(REPLACE(G,'3','')),
		LENGTH(G)-LENGTH(REPLACE(G,'2','')),
		LENGTH(G)-LENGTH(REPLACE(G,'1',''))
	FROM INPUTS)
,SEQ(N) AS (		--SECUENCE 0-N
	SELECT 0
	UNION ALL
	SELECT N+1 FROM SEQ
	WHERE N<(SELECT MAX(MAX(O3),MAX(02),MAX(01)) FROM PINPUTS))
,FIRSTGROUP(G,O3,O2,O1,VF3,VF2,VF1) AS (	--INPUTS AND FIRST POSSIBLE GROUP
	SELECT G,O3,O2,O1,S3.N,S2.N,S1.N
	FROM PINPUTS
	JOIN SEQ S3 ON S3.N<=O3		--CHECK FOR ALL POSSIBLE VALUES OF COUNT1, COUNT2, COUNT3
	JOIN SEQ S2 ON S2.N<=O2
	JOIN SEQ S1 ON S1.N<=O1
	WHERE (O3*3+O2*2+O1)%3=0	--DISCARD INPUTS FROM PINPUTS WHERE TOTAL SUM IS NOT MULTIPLE OF 3
	AND (S3.N*3+S2.N*2+S1.N)=(O3*3+O2*2+O1)/3	--COUNTS1-3 MUST SUM TOTAL/3
	AND ((O3>0 AND S3.N>0) OR (O3=0 AND O2>0 AND S2.N>0) OR (O3=0 AND O2=0))) --OPTIONAL (FOR PERFOMANCE AND ORDER): GROUP MUST HAVE AT LEAST ONE OF THE BIGGEST VALUE
,SECONDGROUP(G,O3,O2,O1,VF3,VF2,VF1,VS3,VS2,VS1) AS (	--INPUTS, FIRST, AND SECOND POSSIBLE GROUP. IF TWO GROUPS EXISTS, THE THIRD WILL TOO
	SELECT G,O3,O2,O1,VF3,VF2,VF1,S3.N,S2.N,S1.N
	FROM FIRSTGROUP
	JOIN SEQ S3 ON S3.N<=(O3-VF3)		--CHECK FOR ALL POSSIBLE VALUES OF COUNT1, COUNT2, COUNT3
	JOIN SEQ S2 ON S2.N<=(O2-VF2)
	JOIN SEQ S1 ON S1.N<=(O1-VF1)
	WHERE (S3.N*3+S2.N*2+S1.N)=(O3*3+O2*2+O1)/3	--COUNTS1-3 MUST SUM TOTAL/3
	AND (((O3-VF3)>0 AND S3.N>0) OR ((O3-VF3)=0 AND (O2-VF2)>0 AND S2.N>0) OR ((O3-VF3)=0 AND (O2-VF2)=0)) --OPTIONAL (FOR PERFOMANCE AND ORDER): GROUP MUST HAVE AT LEAST ONE OF THE BIGGEST VALUE
	AND (S3.N*1000000+S2.N*1000+S1.N)<=(VF3*1000000+VF2*1000+VF1) --OPTIONAL (FOR KEEPING ORDER AND AVOID DUPLICATES):
),THIRDGROUP(G,O3,O2,O1,VF3,VF2,VF1,VS3,VS2,VS1,VT3,VT2,VT1) AS (	--OPTIONAL: INPUTS AND POSSIBLE GROUPS
	SELECT G,O3,O2,O1,VF3,VF2,VF1,VS3,VS2,VS1,(O3-VF3-VS3),(O2-VF2-VS2),(O1-VF1-VS1)
	FROM SECONDGROUP
	WHERE ((O3-VF3-VS3)*1000000+(O2-VF2-VS2)*1000+(O1-VF1-VS1))<=(VS3*1000000+VS2*1000+VS1) --OPTIONAL (FOR KEEPING ORDER AND AVOID DUPLICATES):
)

SELECT INPUTS.G || ' ' || CASE WHEN THIRDGROUP.G IS NULL THEN 'No' ELSE 'Yes   ' ||
CASE WHEN VF3>0 THEN '3' ELSE '' END || CASE WHEN VF3>1 THEN '*' || VF3 ELSE '' END ||
CASE WHEN VF2>0 THEN CASE WHEN VF3>0 THEN '+' ELSE '' END ||'2' ELSE '' END || CASE WHEN VF2>1 THEN '*' || VF2 ELSE '' END ||
CASE WHEN VF1>0 THEN CASE WHEN VF3>0 OR VF2>0 THEN '+' ELSE '' END ||'1' ELSE '' END || CASE WHEN VF1>1 THEN '*' || VF1 ELSE '' END
|| ' , ' ||
CASE WHEN VS3>0 THEN '3' ELSE '' END || CASE WHEN VS3>1 THEN '*' || VS3 ELSE '' END ||
CASE WHEN VS2>0 THEN CASE WHEN VS3>0 THEN '+' ELSE '' END ||'2' ELSE '' END || CASE WHEN VS2>1 THEN '*' || VS2 ELSE '' END ||
CASE WHEN VS1>0 THEN CASE WHEN VS3>0 OR VS2>0 THEN '+' ELSE '' END ||'1' ELSE '' END || CASE WHEN VS1>1 THEN '*' || VS1 ELSE '' END
|| ' , ' ||
CASE WHEN VT3>0 THEN '3' ELSE '' END || CASE WHEN VT3>1 THEN '*' || VT3 ELSE '' END ||
CASE WHEN VT2>0 THEN CASE WHEN VT3>0 THEN '+' ELSE '' END ||'2' ELSE '' END || CASE WHEN VT2>1 THEN '*' || VT2 ELSE '' END ||
CASE WHEN VT1>0 THEN CASE WHEN VT3>0 OR VT2>0 THEN '+' ELSE '' END ||'1' ELSE '' END || CASE WHEN VT1>1 THEN '*' || VT1 ELSE '' END
END
FROM INPUTS
LEFT JOIN THIRDGROUP ON INPUTS.G=THIRDGROUP.G;
```