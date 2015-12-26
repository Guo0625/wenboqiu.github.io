---
layout: post
title: "A*寻路"
date: 2015-06-06 07:05:11 +0800
comments: true
categories: 
---

这几天在公司做一个2D RPG游戏的demo，涉及到寻路的问题，于是自己写了A*寻路算法，分享一下。

地图初始化时，只需要告诉PathFind这个类地图的宽度跟高度，并且初始化障碍点即可。
{% codeblock lang:cpp %}
PathFind::InitMap(_tileMap->getMapSize().width, _tileMap->getMapSize().height);

for (int i=0; i<_tileMap->getMapSize().width; ++i)
{
	for (int j=0; j<_tileMap->getMapSize().height; ++j)
	{
		if (!isWalkable(Vec2(i,j)))
		{
			PathFind::UpdateMapPoint(i, j, false);
		}
	}
}
{% endcodeblock %}
使用时，只需要调用FindPath方法，设置能够行走的方向（八方向or四方向），是否无视边角。该方法返回寻路是否成功。如果成功，路径数据会被存入栈中m_pathList，调用NextPathPoint方法，即可获得下一个目标点。
{% codeblock lang:cpp %}
m_pathFind.FindPath(start, end, m_directionMode==kDirectionMode8?true:false)
m_pathFind.NextPathPoint()
{% endcodeblock %}
源码在最下面--->>

参考文章：

1. 理解A*寻路算法具体过程 (http://www.cnblogs.com/technology/archive/2011/05/26/2058842.html)

2. A* Pathfinding for Beginners (http://www.policyalmanac.org/games/aStarTutorial.htm)


另外提一下，网上热传的一个45度ARPG demo <<热血沙城>>，我下载了它的Cocos2d-x2.2.5 C++修改版，虽然作者说该demo使用了A*寻路，但是实际运行起来，效率很低，一个稍微需要绕道的路径，电脑能够卡好几秒，且人物走的并不是最短路径，更像是广度优先搜索的结果。
(http://blog.csdn.net/zym_123456/article/details/42641591)

<!--more-->
PathFind.h--->>
{% codeblock lang:cpp %}
#ifndef __PATHFIND_H__
#define __PATHFIND_H__

#include <vector>
#include <stack>
#include <algorithm> // for std::find
#include "cocos2d.h"

class PathPoint
{
public:
	PathPoint(int x, int y, bool isWalkable);
	~PathPoint();

	void ClearPathInfo();

	inline int GetX(){return m_x;}
	inline int GetY() {return m_y;}
	
	inline float GetPathFValue(){return m_FValue;}
	inline void SetPathGValue(float gValue){m_GValue = gValue; m_FValue=m_GValue+m_HValue;}
	inline float GetPathGValue(){return m_GValue;}
	inline void SetPathHValue(float hValue){m_HValue = hValue; m_FValue=m_GValue+m_HValue;}
	inline float GetPathHValue(){return m_HValue;}

	inline void SetWalkable(bool isWalkable){m_bWalkable = isWalkable;}
	inline bool GetWalkable(){return m_bWalkable;}

	PathPoint* m_parentPoint;

private:
	int m_x;
	int m_y;
	bool m_bWalkable;
	float m_FValue;
	float m_GValue;
	float m_HValue;
};

class PathFind
{
public:
	PathFind():m_pathEnd(cocos2d::Vec2(-1,-1)){}
	~PathFind(){}

	static void InitMap(int mapWidth, int mapHeight);
	static bool UpdateMapPoint(int x, int y, bool isWalkAble);

	bool FindPath(cocos2d::Point start, cocos2d::Point end, bool is8Direction, bool isIgnoreCorner = false);
	void PrintPath();
	cocos2d::Point NextPathPoint();
	inline cocos2d::Point GetPathEnd(){return m_pathEnd;}

protected:
	bool FindPath(PathPoint* start, PathPoint* end, bool is8Direction, bool isIgnoreCorner = false);
	void GeneratePath(PathPoint* end);

	std::vector<PathPoint*> SurroundPoints(PathPoint* center, bool is8Direction, bool isIgnoreCorner = false);
	PathPoint* MinPoint();
	bool IsWalkable(cocos2d::Vec2 tileCoord);
	void AddSurroundPoint(std::vector<PathPoint*>& surroundPoints, cocos2d::Vec2 tileCoord);

	void FoundPoint(PathPoint* center, PathPoint* surround);
	void NotFoundPoint(PathPoint* center, PathPoint* end, PathPoint* surround);

	float CalculateGValue(PathPoint* start, PathPoint* end);
	float CalculateHValue(PathPoint* start, PathPoint* end);

private:
	std::vector<PathPoint*> m_openList;
	std::vector<PathPoint*> m_closeList;

	std::stack<cocos2d::Point> m_pathList;

	cocos2d::Point m_pathEnd;
};

#endif // __PATHFIND_H__
{% endcodeblock %}

PathFind.cpp --->>

{% codeblock lang:cpp %}
#include "PathFind.h"

USING_NS_CC;

PathPoint::PathPoint(int x, int y, bool isWalkable)
	:m_x(x)
	,m_y(y)
	,m_bWalkable(isWalkable)
	,m_FValue(0.0f)
	,m_GValue(0.0f)
	,m_HValue(0.0f)
	,m_parentPoint(NULL)
{
}

PathPoint::~PathPoint()
{
}

void PathPoint::ClearPathInfo()
{
	m_FValue = 0.0f;
	m_GValue = 0.0f;
	m_HValue = 0.0f;
	m_parentPoint = NULL;
}

static std::vector<std::vector<PathPoint>> m_mapList;
static int m_mapWidth = 0;
static int m_mapHeight = 0;
static bool m_bMapHasInit = false;

void PathFind::InitMap(int width, int height)
{
	m_mapList.clear();
	m_mapList.reserve(width);
	for (int i=0; i<width; ++i)
	{
		std::vector<PathPoint> column;
		m_mapList.reserve(height);

		for (int j=0; j<height; ++j)
		{
			PathPoint point = PathPoint(i,j,true);
			column.push_back(point);
		}

		m_mapList.push_back(column);
	}

	m_mapWidth = width;
	m_mapHeight = height;
	m_bMapHasInit = true;
}

bool PathFind::UpdateMapPoint(int x, int y, bool isWalkAble)
{
	if (!m_bMapHasInit || x >= m_mapWidth || y >= m_mapHeight)
	{
		return false;
	}

	m_mapList[x][y].SetWalkable(isWalkAble);
	return true;
}

bool PathFind::FindPath(Point start, Point end, bool is8Direction, bool isIgnoreCorner)
{
	if (m_bMapHasInit
		&& start.x>=0 && start.x < m_mapWidth && start.y>=0 && start.y < m_mapHeight 
		&& end.x>=0 && end.x < m_mapWidth && end.y>=0 && end.y < m_mapHeight )
	{
		return FindPath(&m_mapList[start.x][start.y], &m_mapList[end.x][end.y], is8Direction, isIgnoreCorner);
	}else{
		return false;
	}
}

bool PathFind::FindPath(PathPoint* start, PathPoint* end, bool is8Direction, bool isIgnoreCorner)
{
	m_closeList.clear();
	m_openList.clear();

	start->ClearPathInfo();
	m_openList.push_back(start);

	while (!m_openList.empty())
	{
		PathPoint* tempStart = MinPoint();
		m_closeList.push_back(tempStart);

		std::vector<PathPoint*> surroundPoints = SurroundPoints(tempStart, is8Direction, isIgnoreCorner);

		std::vector<PathPoint*>::iterator iter = surroundPoints.begin();

		for (; iter!=surroundPoints.end(); ++iter)
		{
			if (std::find(m_openList.begin(), m_openList.end(), *iter) != m_openList.end())
			{
				FoundPoint(tempStart, *iter);
			}else
			{
				NotFoundPoint(tempStart, end, *iter);
			}
		}

		if (std::find(surroundPoints.begin(), surroundPoints.end(), end) != surroundPoints.end())
		{
			GeneratePath(end);
			m_pathEnd = Point(end->GetX(), end->GetY());
			return true;
		}
	}

	return false;
}

void PathFind::GeneratePath(PathPoint* end)
{
	while (!m_pathList.empty())
	{
		m_pathList.pop();
	}

	PathPoint* point = end;

	while (point->m_parentPoint)
	{
		m_pathList.push(Point(point->GetX(), point->GetY()));
		point = point->m_parentPoint;
	}
}

std::vector<PathPoint*> PathFind::SurroundPoints(PathPoint* center, bool is8Direction, bool isIgnoreCorner)
{
	std::vector<PathPoint*> surroundPoints;

	Vec2 relatives[4] = {Vec2(0,-1), Vec2(-1,0), Vec2(1,0), Vec2(0,1)};
	Vec2 coords[4];

	for (int i = 0; i < 4; ++i)
	{
		coords[i] = ccpAdd(Vec2(center->GetX(), center->GetY()), relatives[i]);
		AddSurroundPoint(surroundPoints, coords[i]);
	}

	if (is8Direction)
	{
		Vec2 obliquedRelatives[4] = {Vec2(-1,-1),Vec2(1,-1),Vec2(-1,1),Vec2(1,1)};
		
		bool flag[4];
		Vec2 obliquedCoords[4];

		for (int i=0; i<4; ++i)
		{
			obliquedCoords[i] = ccpAdd(Vec2(center->GetX(), center->GetY()), obliquedRelatives[i]);
			flag[i] = true;
		}

		if (!isIgnoreCorner)
		{
			if (!IsWalkable(coords[0]))
			{
				flag[0]=flag[1]=false;
			}

			if (!IsWalkable(coords[1]))
			{
				flag[0]=flag[2]=false;
			}

			if (!IsWalkable(coords[2]))
			{
				flag[1]=flag[3]=false;
			}

			if (!IsWalkable(coords[3]))
			{
				flag[2]=flag[3]=false;
			}
		}

		for (int i=0; i<4; ++i)
		{
			if (flag[i])
			{
				AddSurroundPoint(surroundPoints, obliquedCoords[i]);
			}
		}
		
	}

	return surroundPoints;
}

PathPoint* PathFind::MinPoint()
{
	if (m_openList.size()>0) {
		PathPoint* minPoint = m_openList.at(0);
		int minIndex = 0;

		for (int i=0; i<m_openList.size(); ++i)
		{
			if (m_openList.at(i)->GetPathFValue() < minPoint->GetPathFValue())
			{
				minPoint = m_openList.at(i);
				minIndex = i;
			}
		}

		auto it = std::next( m_openList.begin(), minIndex );
		m_openList.erase(it);

		return minPoint;
	}
	else
	{
		return NULL;
	}
}

bool PathFind::IsWalkable(Vec2 tileCoord)
{
	if (tileCoord.x < 0 || tileCoord.x >= m_mapWidth || tileCoord.y < 0 || tileCoord.y >= m_mapHeight)
	{
		return false;
	}

	if (m_mapList[tileCoord.x][tileCoord.y].GetWalkable())
	{
		return true;
	}

	return false;
}

void PathFind::AddSurroundPoint(std::vector<PathPoint*>& surroundPoints, Vec2 tileCoord)
{
	if (IsWalkable(tileCoord))
	{
		if (std::find(m_closeList.begin(), m_closeList.end(), &m_mapList[tileCoord.x][tileCoord.y]) != m_closeList.end())
		{
			return;
		}

		surroundPoints.push_back(&m_mapList[tileCoord.x][tileCoord.y]);
	}
}

void PathFind::FoundPoint(PathPoint* center, PathPoint* surround)
{
	float oldGValue = surround->GetPathGValue();
	float newGValue = center->GetPathGValue() + CalculateGValue(center, surround);

	if (oldGValue > newGValue)
	{
		surround->m_parentPoint = center;
		surround->SetPathGValue(newGValue);
	}
}

void PathFind::NotFoundPoint(PathPoint* center, PathPoint* end, PathPoint* surround)
{
	float newGValue = center->GetPathGValue() + CalculateGValue(center, surround);
	surround->SetPathGValue(newGValue);
	surround->SetPathHValue(CalculateHValue(surround, end));

	surround->m_parentPoint = center;
	m_openList.push_back(surround);
}

float PathFind::CalculateGValue(PathPoint* start, PathPoint* end)
{
	float _offsetX = start->GetX() - end->GetX();
	float _offsetY = start->GetY() - end->GetY();
	return sqrt( _offsetX * _offsetX + _offsetY * _offsetY);
}

float PathFind::CalculateHValue(PathPoint* start, PathPoint* end)
{
	float _x = abs(start->GetX() - end->GetX());
	float _y = abs(start->GetY() - end->GetY());

	return _x + _y;
}

void PathFind::PrintPath()
{
	if (!m_pathList.empty())
	{
		CCLOG("Path Start------------------------------------------------------------");
		//int step = 1;
		//for (std::vector<Point>::iterator iter=m_pathList.begin(); iter!=m_pathList.end(); ++iter)
		//{
		//	CCLOG("Step %d:(%d,%d)", step, (*iter).x, (*iter).y);
		//	++step;
		//}
		CCLOG("Path End--------------------------------------------------------------");
	}
}

Point PathFind::NextPathPoint()
{
	if (!m_pathList.empty())
	{
		Point dest = m_pathList.top();
		m_pathList.pop();
		return dest;
	}
	
	return Point(-1,-1);
}
{% endcodeblock %}






