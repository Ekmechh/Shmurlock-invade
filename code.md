/*
 * GameCanvas.cpp
 *
 *  Created on: October 16, 2021
 *      Author: Ekmech
 */


#include "GameCanvas.h"

const int GameCanvas::ENEMYANIMATION_WALK = 0;
const int GameCanvas::ENEMYANIMATION_ATTACK = 1;
const int GameCanvas::ENEMYANIMATION_DEATH = 2;


GameCanvas::GameCanvas(gApp* root) : gBaseCanvas(root) {
	this->root = root;
}

GameCanvas::~GameCanvas() {
}

void GameCanvas::setup() {
//	gLogi("GameCanvas") << "setup";
	background.loadImage("oyun/haritalar/arkaplan1.jpg");
	for(int i = 0; i < characterframenum; i++) character[i].loadImage("oyun/karakterler/erkek/erkek_tufek_0" + gToStr(i) + ".png");
	enemyframenum[ENEMYANIMATION_WALK] = 8;
	enemyframenum[ENEMYANIMATION_ATTACK] = 16;
	enemyframenum[ENEMYANIMATION_DEATH] = 14;
	for(int i = 0; i < enemyanimationnum; i++) {
		std::string animpath = "walk/Walk";
		if(i == ENEMYANIMATION_ATTACK) animpath = "attack/attack1";
		else if(i == ENEMYANIMATION_DEATH) animpath = "death/Death";
		for(int j = 0; j < enemyframenum[i]; j++) {
			std::string fno = gToStr(j);
			if(j < 10) fno = "0" + fno;
			enemyimage[i][j].loadImage("oyun/dusmanlar/" + animpath + "_0" + fno + ".png");
		}
	}
	levelmapimage.loadImage("oyun/haritalar/radar1.jpg");
	levelmapsign1.loadImage("oyun/haritalar/radarisaret1.png");
	levelmapsign2.loadImage("oyun/haritalar/radarisaret2.png");
	bulletimage[BULLETSENDER_CHARACTER].loadImage("oyun/objeler/mermi.png");
	bulletimage[BULLETSENDER_ENEMY].loadImage("oyun/objeler/mermi2.png");
	cx = (getWidth() - character[0].getWidth()) / 2;
	cy = (getHeight() - character[0].getHeight()) / 2;
	crot = 0.0f;
	keystate = KEY_NONE;
	cdx = 0.0f;
	cdy = 0.0f;
	cspeed = 4.0f;
	camx = 0;
	camy = 0;
	camleftmargin = getWidth() / 4;
	camtopmargin = getHeight() / 4;
	camrightmargin = (getWidth() * 3) / 4;
	cambottommargin = (getHeight() * 3) / 4;
	characterframeno = 0;
	characterframecounter = 0;
	characterframecounterlimit = 6;
	enemynum = 20;
	enemyframecounterlimit = 3;
	for(int i = 0; i < enemynum; i++) {
		Enemy e;
		enemy.push_back(e);
		int ex, ey;
		do {
			ex = gRandom(background.getWidth() - enemyimage[0][0].getWidth());
			ey = gRandom(background.getHeight() - enemyimage[0][0].getHeight());
		} while(ex < getWidth() && ey < getHeight());
		enemy[i].setPosition(ex, ey);

		enemy[i].setRotation(0.0f);
		enemy[i].setFrameNo(gRandom(enemyframenum[ENEMYANIMATION_WALK]));
		enemy[i].setFrameCounter(gRandom(enemyframecounterlimit));
		enemy[i].setAnimationNo(ENEMYANIMATION_WALK);
	}
	levelmapx = getWidth() - levelmapimage.getWidth() - (levelmapimage.getWidth() / 2);
	levelmapy = levelmapimage.getWidth() / 2;
	bulletspeed = cspeed * 6;
	muzzleangle = gRadToDeg(std::atan2(2 - (character[0].getHeight() / 2), 43 - (character[0].getWidth() / 2))) + 90.0f;
	muzzledistance = std::sqrt(std::pow(2 - (character[0].getHeight() / 2), 2) + std::pow(43 - (character[0].getWidth() / 2), 2));
	deadenemynum = 0;
	enemyrighthandangle = gRadToDeg(std::atan2(109 - (enemyimage[0][0].getHeight() / 2), 58 - (enemyimage[0][0].getWidth() / 2))) - 90.0f;
	enemyrighthanddistance = std::sqrt(std::pow(109 - (enemyimage[0][0].getHeight() / 2), 2) + std::pow(58 - (enemyimage[0][0].getWidth() / 2), 2));
	enemylefthandangle = gRadToDeg(std::atan2(106 - (enemyimage[0][0].getHeight() / 2), 91 - (enemyimage[0][0].getWidth() / 2))) - 90.0f;
	enemylefthanddistance = std::sqrt(std::pow(106 - (enemyimage[0][0].getHeight() / 2), 2) + std::pow(91 - (enemyimage[0][0].getWidth() / 2), 2));
	cinitiallive = 100;
	clive = cinitiallive;
}

void GameCanvas::update() {
//	gLogi("GameCanvas") << "update";
	moveCharacter();
	moveCamera();
	moveEnemies();
	moveBullets();
}

void GameCanvas::draw() {
//	gLogi("GameCanvas") << "draw";
	drawBackground();
	drawEnemies();
	drawCharacter();
	drawBullets();
	drawLevelMap();
}

void GameCanvas::moveCharacter() {
	if((keystate & KEY_W) != 0) {
		cdx = std::sin(gDegToRad(crot)) * cspeed;
		cdy = -std::cos(gDegToRad(crot)) * cspeed;
	} else if((keystate & KEY_S) != 0) {
		cdx = -std::sin(gDegToRad(crot)) * cspeed;
		cdy = std::cos(gDegToRad(crot)) * cspeed;
	}

	if((keystate & KEY_D) != 0) {
		cdx = std::cos(gDegToRad(crot)) * cspeed;
		cdy = std::sin(gDegToRad(crot)) * cspeed;
	} else if((keystate & KEY_A) != 0) {
		cdx = -std::cos(gDegToRad(crot)) * cspeed;
		cdy = -std::sin(gDegToRad(crot)) * cspeed;
	}

	cx += cdx;
	cy += cdy;

	if(cdx != 0.0f || cdy != 0.0f) {
		characterframecounter++;
		if(characterframecounter >= characterframecounterlimit) {
			characterframeno++;
			if(characterframeno >= characterframenum) characterframeno = 0;
			characterframecounter = 0;
		}
	}
}

void GameCanvas::moveCamera() {
	camleftmargin = getWidth() / 4;
	if(camx <= 0) camleftmargin = 0;
	camtopmargin = getHeight() / 4;
	if(camy <= 0) camtopmargin = 0;
	camrightmargin = (getWidth() * 3) / 4;
	if(camx + getWidth() >= background.getWidth()) camrightmargin = getWidth();
	cambottommargin = (getHeight() * 3) / 4;
	if(camy + getHeight() >= background.getHeight()) cambottommargin = getHeight();

	if(cx <= camleftmargin || cx + character[0].getWidth() >= camrightmargin) {
		cx -= cdx;
		camx += cdx;
		if(camx <= 0) {
			camx = 0;
		}
		if(camx + getWidth() >= background.getWidth()) {
			camx = background.getWidth() - getWidth();
		}
	}
	if(cy <= camtopmargin || cy + character[0].getHeight() >= cambottommargin) {
		cy -= cdy;
		camy += cdy;
		if(camy <= 0) {
			camy = 0;
		}
		if(camy + getHeight() >= background.getHeight()) {
			camy = background.getHeight() - getHeight();
		}
	}

	cdx = 0.0f;
	cdy = 0.0f;
}

void GameCanvas::moveEnemies() {
	for(int i = 0; i < enemynum; i++) {
		if(enemy[i].getAnimationNo() != ENEMYANIMATION_DEATH) enemy[i].setRotation(gRadToDeg(std::atan2((cy + camy + character[0].getHeight() / 2) - (enemy[i].getY() + enemyimage[0][0].getHeight() / 2), (cx + camx + character[0].getWidth() / 2) - (enemy[i].getX() + enemyimage[0][0].getWidth() / 2))) - 90.0f);

		float edx = 0.0f;
		float edy = 0.0f;
		if(enemy[i].getAnimationNo() == ENEMYANIMATION_WALK && enemy[i].getX() + enemyimage[0][0].getWidth() > camx && enemy[i].getX() < camx + getWidth() && enemy[i].getY() + enemyimage[0][0].getHeight() > camy && enemy[i].getY() < camy + getHeight()) {
			edx = -std::sin(gDegToRad(enemy[i].getRotation()));
			edy = std::cos(gDegToRad(enemy[i].getRotation()));
		}

		enemy[i].setPosition(enemy[i].getX() + edx, enemy[i].getY() + edy);

		if(enemy[i].getAnimationNo() != ENEMYANIMATION_DEATH && checkCollision(enemy[i].getX() + 49, enemy[i].getY() + 25, enemy[i].getX() + 114, enemy[i].getY() + 70,
				cx + camx, cy + camy + 30, cx + camx + character[0].getWidth(), cy + camy + character[0].getHeight())) {
			if(enemy[i].getAnimationNo() != ENEMYANIMATION_ATTACK) enemy[i].setFrameNo(0);
			enemy[i].setAnimationNo(ENEMYANIMATION_ATTACK);
			if(enemy[i].getFrameNo() == 2 || enemy[i].getFrameNo() == 9) {
				clive -= 4;
				if(clive <= 0) {
					clive = 0;
					gLogi("GameCanvas") << "OYUNU KAYBETTINIZ 2";
				}
			}

		} else {
			if(enemy[i].getAnimationNo() != ENEMYANIMATION_DEATH) {
				if(enemy[i].getAnimationNo() != ENEMYANIMATION_WALK) enemy[i].setFrameNo(0);
				enemy[i].setAnimationNo(ENEMYANIMATION_WALK);
			}
		}

		if((edx != 0.0f || edy != 0.0f)  || enemy[i].getAnimationNo() != ENEMYANIMATION_WALK) {
			enemy[i].setFrameCounter(enemy[i].getFrameCounter() + 1);
			if(enemy[i].getFrameCounter() >= enemyframecounterlimit) {
				enemy[i].setFrameNo(enemy[i].getFrameNo() + 1);
				if(enemy[i].getFrameNo() >= enemyframenum[enemy[i].getAnimationNo()]) {
					if(enemy[i].getAnimationNo() == ENEMYANIMATION_DEATH) enemy[i].setFrameNo(enemyframenum[enemy[i].getAnimationNo()] - 1);
					else enemy[i].setFrameNo(0);
				}
				enemy[i].setFrameCounter(0);

				if(enemy[i].getAnimationNo() == ENEMYANIMATION_WALK && (enemy[i].getFrameNo() == 3 || enemy[i].getFrameNo() == 7)) {
					float bx, by;
					if(enemy[i].getFrameNo() == 3) {
						bx = enemy[i].getX() + ((enemyimage[0][0].getWidth() - bulletimage[BULLETSENDER_ENEMY].getWidth()) / 2) - (std::sin(gDegToRad(enemy[i].getRotation() + enemyrighthandangle)) * enemyrighthanddistance);
						by = enemy[i].getY() + ((enemyimage[0][0].getHeight() - bulletimage[BULLETSENDER_ENEMY].getHeight()) / 2) + (std::cos(gDegToRad(enemy[i].getRotation() + enemyrighthandangle)) * enemyrighthanddistance);
					} else {
						bx = enemy[i].getX() + ((enemyimage[0][0].getWidth() - bulletimage[BULLETSENDER_ENEMY].getWidth()) / 2) - (std::sin(gDegToRad(enemy[i].getRotation() + enemylefthandangle)) * enemylefthanddistance);
						by = enemy[i].getY() + ((enemyimage[0][0].getHeight() - bulletimage[BULLETSENDER_ENEMY].getHeight()) / 2) + (std::cos(gDegToRad(enemy[i].getRotation() + enemylefthandangle)) * enemylefthanddistance);
					}
					float bdx = -std::sin(gDegToRad(enemy[i].getRotation())) * bulletspeed;
					float bdy = std::cos(gDegToRad(enemy[i].getRotation())) * bulletspeed;
					generateBullet(bx, by, bdx, bdy, enemy[i].getRotation(), BULLETSENDER_ENEMY);
				}
			}
		}
	}
}

void GameCanvas::moveBullets() {
	for(int i = bullets.size() - 1; i >= 0; i--) {
		bullets[i][0] += bullets[i][2];
		bullets[i][1] += bullets[i][3];
		bullets[i][5]++;
		if(bullets[i][5] >= 30) {
			bullets.erase(bullets.begin() + i);
			continue;
		}

		if(bullets[i][0] + bulletimage[0].getWidth() < camx || bullets[i][0] >= camx + getWidth() || bullets[i][1] + bulletimage[0].getHeight() < camy || bullets[i][1] >= camy + getHeight()) {
			bullets.erase(bullets.begin() + i);
			continue;
		}

		if(bullets[i][6] == BULLETSENDER_CHARACTER) {
			for(int j = 0; j < enemynum; j++) {
				if(enemy[j].getAnimationNo() != ENEMYANIMATION_DEATH && checkCollision(bullets[i][0], bullets[i][1], bullets[i][0] + bulletimage[0].getWidth(), bullets[i][1] + bulletimage[0].getHeight(),
						enemy[j].getX(), enemy[j].getY(), enemy[j].getX() + enemyimage[0][0].getWidth(), enemy[j].getY() + enemyimage[0][0].getHeight())) {
					enemy[j].setAnimationNo(ENEMYANIMATION_DEATH);
					enemy[j].setFrameNo(0);
					deadenemynum++;
					//TODO olen enemy vektorun bas�na kaydedilecek
					bullets.erase(bullets.begin() + i);
					break;
				}
			}
		} else if(bullets[i][6] == BULLETSENDER_ENEMY) {
			if(checkCollision(bullets[i][0], bullets[i][1], bullets[i][0] + bulletimage[0].getWidth(), bullets[i][1] + bulletimage[0].getHeight(),
						cx + camx, cy + camy, cx + camx + character[0].getWidth(), cy + camy + character[0].getHeight())) {
				clive--;
				if(clive <= 0) {
					clive = 0;
					gLogi("GameCanvas") << "OYUNU KAYBETTINIZ 1";
				}
				bullets.erase(bullets.begin() + i);
				break;
			}
		}
	}
}

void GameCanvas::drawBackground() {
	background.drawSub(0, 0, getWidth(), getHeight(), camx, camy, getWidth(), getHeight());
}

void GameCanvas::drawCharacter() {
	character[characterframeno].draw(cx, cy, character[0].getWidth(), character[0].getHeight(), crot);
}

void GameCanvas::drawEnemies() {
	for(int i = 0; i < enemynum; i++) {
		enemyimage[enemy[i].getAnimationNo()][enemy[i].getFrameNo()].draw(enemy[i].getX() - camx, enemy[i].getY() - camy, enemyimage[0][0].getWidth(), enemyimage[0][0].getHeight(), enemy[i].getRotation());
	}
}

void GameCanvas::drawBullets() {
	for(int i = 0; i < bullets.size(); i++) {
		bulletimage[(int)bullets[i][6]].draw(bullets[i][0] - camx, bullets[i][1] - camy, bulletimage[0].getWidth(), bulletimage[0].getHeight(), bullets[i][4]);
	}
}

void GameCanvas::drawLevelMap() {
	levelmapimage.draw(levelmapx, levelmapy);
	for(int i = 0; i < enemynum; i++) {
		if(enemy[i].getAnimationNo() != ENEMYANIMATION_DEATH) levelmapsign2.draw(levelmapx + 2 + enemy[i].getX() / 32, levelmapy + 2 + enemy[i].getY() / 32);
	}
	levelmapsign1.draw(levelmapx + 2 + (cx + camx) / 32, levelmapy + 2 + (cy + camy) / 32);
}

void GameCanvas::keyPressed(int key) { // w:87 s:83 d:68 a:65
//	gLogi("GameCanvas") << "keyPressed:" << key;
	int pressedkey = KEY_NONE;
	switch(key) {
	case 87:
		pressedkey = KEY_W;
		break;
	case 83:
		pressedkey = KEY_S;
		break;
	case 68:
		pressedkey = KEY_D;
		break;
	case 65:
		pressedkey = KEY_A;
		break;
	default:
		break;
	}
	keystate |= pressedkey;
}

void GameCanvas::keyReleased(int key) {
//	gLogi("GameCanvas") << "keyReleased:" << key;
	int pressedkey = KEY_NONE;
	switch(key) {
	case 87:
		pressedkey = KEY_W;
		break;
	case 83:
		pressedkey = KEY_S;
		break;
	case 68:
		pressedkey = KEY_D;
		break;
	case 65:
		pressedkey = KEY_A;
		break;
	default:
		break;
	}
	keystate &= ~pressedkey;
}

void GameCanvas::mouseMoved(int x, int y) {
//	gLogi("GameCanvas") << "mouseMoved" << ", x:" << x << ", y:" << y;
	crot = gRadToDeg(std::atan2(y - (cy + (character[0].getHeight() / 2)), x - (cx + (character[0].getWidth() / 2)))) + 90.0f;
}

void GameCanvas::mouseDragged(int x, int y, int button) {
//	gLogi("GameCanvas") << "mouseDragged" << ", x:" << x << ", y:" << y << ", b:" << button;
}

void GameCanvas::mousePressed(int x, int y, int button) {
//	gLogi("GameCanvas") << "mousePressed" << ", x:" << x << ", y:" << y << ", b:" << button << ", crot:" << crot;
}

void GameCanvas::mouseReleased(int x, int y, int button) {
//	gLogi("GameCanvas") << "mouseReleased" << ", button:" << button;
	float bx = cx + camx + ((character[0].getWidth() - bulletimage[0].getWidth()) / 2) + (std::sin(gDegToRad(crot + muzzleangle)) * muzzledistance);
	float by = cy + camy + ((character[0].getHeight() - bulletimage[0].getHeight()) / 2) - (std::cos(gDegToRad(crot + muzzleangle)) * muzzledistance);
	float bdx = std::sin(gDegToRad(crot)) * bulletspeed;
	float bdy = -std::cos(gDegToRad(crot)) * bulletspeed;
	generateBullet(bx, by, bdx, bdy, crot, BULLETSENDER_CHARACTER);
}

void GameCanvas::mouseEntered() {
}

void GameCanvas::mouseExited() {
}

void GameCanvas::showNotify() {

}

void GameCanvas::hideNotify() {

}

bool GameCanvas::checkCollision(int xLeft1, int yUp1, int xRight1, int yBottom1, int xLeft2, int yUp2, int xRight2, int yBottom2) {
	if(xLeft1 < xRight2 && xRight1 > xLeft2 && yBottom1 > yUp2 && yUp1 < yBottom2) {
		return true;
	}
	return false;
}

void GameCanvas::generateBullet(float bulletx, float bullety, float bulletdx, float bulletdy, float bulletrot, int bulletSender) {
	std::vector<float> newbullet;
	newbullet.push_back(bulletx);
	newbullet.push_back(bullety);
	newbullet.push_back(bulletdx);
	newbullet.push_back(bulletdy);
	newbullet.push_back(bulletrot);
	newbullet.push_back(0);
	newbullet.push_back(bulletSender);
	bullets.push_back(newbullet);
}
