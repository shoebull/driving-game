# driving-game
package application;

import javafx.animation.AnimationTimer;
import javafx.application.Application;
import javafx.scene.Group;
import javafx.scene.Scene;
import javafx.scene.canvas.Canvas;
import javafx.scene.canvas.GraphicsContext;
import javafx.scene.image.Image;
import javafx.scene.paint.Color;
import javafx.stage.Stage;

public class CarChase extends Application {

    public int CANVAS_SIZE = 800;
    private long lastNanoTime;
    
    private Sprite car;
    private Sprite obstacle;
    private Sprite boost;
    
    private String[] obstacleImages = {"big-truck.png", "hole.png", "police-car.png"};
    
    private Canvas canvas;
    private GraphicsContext gc;
    private Image background;

    // joystick variables
    private Image joystickBase;
    private Image joystickKnob;
    private double centerX = 150, centerY = 650; // Joystick base position
    private double knobX = 150, knobY = 650;     // Current joystick position
    private final double JOYSTICK_RADIUS = 60;
    
    
    private double driftX = 0;
    private double driftTimer = 0;
    
    private double mouseX, mouseY;
    private boolean isGameOver = false;
    
    //alcohol boost variables
    private int alcCount = 0;
    private double effectTimer = 0;
    private final double EFFECT_DURATION = 5.0;
    private boolean isImpaired = false;
    
    

    @Override
    public void start(Stage primaryStage) {

        canvas = new Canvas(CANVAS_SIZE, CANVAS_SIZE);
        gc = canvas.getGraphicsContext2D();

        lastNanoTime = System.nanoTime();

        createSprites();

        new AnimationTimer() {
            public void handle(long currentNanoTime) {
                if (isGameOver) {
                    return;
                }

                double elapsedTime = (currentNanoTime - lastNanoTime) / 1000000000.0;
                lastNanoTime = currentNanoTime;

                updateJoystick();
                updateDrift(elapsedTime);
                applyFriction();
                
                //obstacle logic stuff
                if (car.intersects(obstacle)) {
                    isGameOver = true;
                } else if (obstacle.getYPosition() > CANVAS_SIZE) {
                    resetObstacle();
                }
                
                //alc boost logic
                if(car.intersects(boost)) {
                	alcCount++;
                	isImpaired = true;
                	effectTimer = EFFECT_DURATION;
                	spawnBoost(); 
                } else if (boost.getYPosition() > CANVAS_SIZE + 400) {
                    spawnBoost(); 
                }
                
                if (effectTimer > 0) {
                    effectTimer -= elapsedTime;
                } else {
                    isImpaired = false;
                }

                obstacle.update(elapsedTime);
                car.update(elapsedTime);
                boost.update(elapsedTime);
                
                // stops car from drifting off screen
                keepInBounds(car);

                render();
            }
        }.start();

        Group root = new Group(canvas);
        Scene scene = new Scene(root);

        // track mouse for joystick control
        scene.setOnMouseMoved(e -> {
            mouseX = e.getX();
            mouseY = e.getY();
        });

        primaryStage.setTitle("Impaired Driving");
        primaryStage.setScene(scene);
        primaryStage.show();
    }
    
    
    //STEERING 
    private void updateDrift(double drift) {

        driftTimer -= drift;

        // every few seconds picks a new random drift direction
        if (driftTimer <= 0) {
        	if(isImpaired) {
        		driftX = (Math.random() * 2 - 1) * 700; // last num is strength of drift
        	}else {
        		driftX = (Math.random() * 2 - 1) * 400;
        	}
            
            driftTimer = 1 + Math.random() * 1.5;
        }

        // push car sideways even if player does nothing
        car.addVelocity(driftX * drift, 0);
    }
    
    
    private void applyFriction() {
        double friction; //closer to 1 is more slippery
        
        if(isImpaired) {
        	friction = 0.98;
        }else {
        	friction = 0.97;
        }

        car.setVelocity(
            car.getVelocityX() * friction,
            car.getVelocityY() * friction
        );
    }
    

    private void updateJoystick() {
    	///*
    	//flying box mouse repelling mechanic
    	double dx = knobX - mouseX;
        double dy = knobY - mouseY;
        double distance = Math.hypot(dx, dy);
        

        if (distance < 50) {
            //calculate repulsion angle (from mouse towards knob)
            double angleInRadians = Math.atan2(knobY - mouseY, knobX - mouseX);
            
            double speed = (200 / distance); 
            
            double moveX = Math.cos(angleInRadians) * speed;
            double moveY = Math.sin(angleInRadians) * speed;
            
            knobX += moveX;
            knobY += moveY;
        }
        
        
        //puts a boundary on the joystick so it cant be pushed off the plate
        double distFromCenter = Math.sqrt(Math.pow(knobX - centerX, 2) + Math.pow(knobY - centerY, 2));
        if (distFromCenter > JOYSTICK_RADIUS) {
            double angle = Math.atan2(knobY - centerY, knobX - centerX);
            knobX = centerX + Math.cos(angle) * JOYSTICK_RADIUS;
            knobY = centerY + Math.sin(angle) * JOYSTICK_RADIUS;
        }

        double inputX = (knobX - centerX) / JOYSTICK_RADIUS;
        double inputY = (knobY - centerY) / JOYSTICK_RADIUS;

        car.addVelocity(inputX * 15, inputY * 15);
        
        //unstable steering (higher num = worse control)
        double steerStrength;
        
        if(isImpaired) {
        	steerStrength = 400;
        }else {
        	steerStrength = 200;
        }

        car.addVelocity(
            (inputX * steerStrength) * 0.02,
            (inputY * steerStrength) * 0.02
        );
        
        //pulls knob back to middle (higher multiplier pulls back faster)
        knobX += (centerX - knobX) * 0.01;
        knobY += (centerY - knobY) * 0.01;
        //*/

        
        //OG mouse tracking joystick steering
        /*
        double dx = mouseX - centerX;
        double dy = mouseY - centerY;

        double distance = Math.sqrt(dx * dx + dy * dy);

        double maxRadius = JOYSTICK_RADIUS;

        double nx = dx / distance;
        double ny = dy / distance;

        
        double repelStrength = Math.min(distance, maxRadius);

        double targetX = centerX + nx * repelStrength;
        double targetY = centerY + ny * repelStrength;


        knobX += (targetX - knobX)*0.04;
        knobY += (targetY - knobY)*0.04;

        double inputX = (knobX - centerX) / JOYSTICK_RADIUS;
        double inputY = (knobY - centerY) / JOYSTICK_RADIUS;

        // unstable steering
        double steerStrength = 500;

        car.addVelocity(
            (inputX * steerStrength) * 0.02,
            (inputY * steerStrength) * 0.02
        );
        
        */
    }

    private void render() {
    	gc.drawImage(background, 0, 0, CANVAS_SIZE, CANVAS_SIZE);

        gc.setGlobalAlpha(0.6);
        gc.drawImage(joystickBase,
                centerX - 60,
                centerY - 60,
                120,
                120);

        gc.setGlobalAlpha(1.0);
        gc.drawImage(joystickKnob,knobX - 30, knobY - 30, 60, 60);

        car.render(gc);
        obstacle.render(gc);
        boost.render(gc);
        
        if (isImpaired) {
            gc.setGlobalAlpha(0.3); // transparency
            gc.setFill(Color.BROWN); 
            gc.fillRect(0, 0, CANVAS_SIZE, CANVAS_SIZE);
            gc.setGlobalAlpha(1.0); // Reset alpha so other things aren't transparent
        }
        
        gc.setFill(Color.WHITE); 
        gc.setFont(javafx.scene.text.Font.font("Arial", javafx.scene.text.FontWeight.BOLD, 30));
        gc.fillText("Drinks: " + alcCount, 20, 50);
        
    }

    public void createSprites() {
    	background = new Image(getClass().getResourceAsStream("road.png"));
    	joystickBase = new Image(getClass().getResourceAsStream("joystick-base.png"));
    	joystickKnob = new Image(getClass().getResourceAsStream("knob.png"));
        car = new Sprite();
        
        
        car.setImage("car.png"); 
        car.setPosition(CANVAS_SIZE / 2, CANVAS_SIZE / 2);

        obstacle = new Sprite();
        resetObstacle();
        
        boost = new Sprite();
        boost.setImage("alcohol.png");
        spawnBoost();
        
    }
    
    private void spawnBoost() {
    	double[] lanes = {100, 700};
        
        int randLane = (int)(Math.random() * lanes.length);
        double spawnX = lanes[randLane];
        
        //spawn obstacles 500 above frame so u don't get spawn camped
        double spawnY = -500; 
        double speed = 600;
        
        boost.setPosition(spawnX, spawnY);
        boost.setVelocity(0, speed);
        
        boost.setPosition(spawnX, spawnY);
        boost.setVelocity(0, speed);
    }

    private void resetObstacle() {
    	int randIndex = (int)(Math.random()*obstacleImages.length);
    	String nextImage = obstacleImages[randIndex];
    	
    	obstacle.setImage(nextImage);
    	
        double[] lanes = {200, 500};
        
        int randLane = (int)(Math.random() * lanes.length);
        double spawnX = lanes[randLane];
        
        //spawn obstacles 500 above frame so u don't get spawn camped
        double spawnY = -500; 
        double speed = 250 + (Math.random() * 200);//random speed
        
        
        obstacle.setPosition(spawnX, spawnY);
        obstacle.setVelocity(0, speed);
    }
    
    private void keepInBounds(Sprite s) {
        if (s.getXPosition() < 0) s.setXPosition(0);
        if (s.getXPosition() > CANVAS_SIZE - 50) s.setXPosition(CANVAS_SIZE - 50);
        if (s.getYPosition() < 0) s.setYPosition(0);
        if (s.getYPosition() > CANVAS_SIZE - 50) s.setYPosition(CANVAS_SIZE - 50);
    }

    public static void main(String[] args) {
    	
        launch(args);
    }
}
